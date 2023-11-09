.. title: Debugging UB in `unsafe` Rust Code
.. slug: debugging-ub-unsafe-rust-code
.. date: 2023-11-09 13:45:00 UTC+05:30
.. tags: Rust, Unsafe, UB, SCTP
.. author: Abhijit Gadgil
.. link:
.. description:
.. type:
.. status:
.. summary: This blog post explores how we faced UB in some `unsafe` Rust code and how the issue was finally resolved.

# Introduction

SCTP is a transport protocol standardized by IETF that is widely used in the 3GPP Control protocol specifications. For instance, RAN and Core network application protocols in cellular mobile networks make use of SCTP as a transport protocol. SCTP offers some additional benefits compared to TCP - that include support for multiple streams, multi-homing etc. In one of the open source projects, [`ellora`](https://github.com/gabhijit/ellora/){:target="\_blank"}, I am implementing ergonomic safe Rust APIs for the Linux kernel's SCTP stack, so that it is possible to use SCTP for application development with Rust's `async` ecosystem. While providing these APIs [`libc`](https://docs.rs/libc/latest/libc){:target="\_blank"} crate is heavily used. This crate provides Rust wrappers over `libc`. Lot of these functions require de-referencing or passing raw pointers and are thus `unsafe` in the Rust. Recently, I faced a strange issue - some of the unit tests would fail if run with `--release` flag, the same unit tests would run fine without the `--release` flag. Since we are making use of the `unsafe` code initial hypothesis was, may be there is something we are not doing right in one of the `unsafe` blocks, which turned out to be correct, but actually solving the problem was considerably involved. This blog post discusses the approach followed to troubleshoot the problem. In general, the following discussion should be useful for debugging a running application.

# Overview of the Problem

While upgrading examples in the project [`ellora`](https://github.com/gabhijit/ellora){:target="\_blank"} to use `clap` (a Rust crate for developing CLI programs) `v4`, we observed that when we were running `cargo test --release` a couple of test cases were failing, but the same test cases worked just fine if we were running `cargo test` (ie. in the `debug` mode). One of the `SCTP` API methods, `sctp_send` on the `SctpListener` socket was failing with an error value of `EINVAL`. What was really odd was - if we added some logging around the failing code, the test cases would succeed, which made the problem even more interesting to debug because simply adding some logging around the failing part didn't help at all.

This was happening inside an `unsafe` block, that suggested, something in the code was messing up with the `stack-frame` of the function when compiled with `--release` optimizations. Surprisingly, those optimizations didn't hurt when added debug prints or logging. Really odd, but the symptoms were clearly suggesting that there is some *Undefined Behavior (UB)* that we are encountering that is causing this. Also, the code contained assigning a few raw pointers and de-referencing those pointers, very likely some part of that code was a culprit. (Note: while this implementation uses `unsafe` code, the APIs exposed are completely safe, idiomatic Rust API so this is part of internal implementation detail and no user facing code can directly call these functions.)

What we are basically doing is making use of a low-level [`AsyncFd`](https://docs.rs/tokio/latest/tokio/io/unix/struct.AsyncFd.html){:target="\_blank"} API from [`tokio`](https://docs,rs/tokio/latest/tokio){:target="\_blank"} and then using the underlying file descriptor (`RawFd`) for performing the actual I/O. The underlying `RawFd` is set to non-blocking upon creation and registered with `epoll`. Thus it can be used through `epoll` based reactors. That's the high level idea, there are some details, but they are not quite relevant for the current discussion. (Additional note: when we will fully support _any_ `async` runtime this `AsyncFd` should be abstracted out.)

The implementation uses `sendmsg` system call and uses control message to send any optional send side or receive side information. The API is very similar to [`sctplib`]'s `sctp_send` [api](https://github.com/sctp/lksctp-tools/blob/master/man/sctp_send.3){:target="\_blank"}.

If you have some heart for unsafe code, this is what the code looks like. The actual code listing can be [found here](https://github.com/gabhijit/ellora/blob/main/sctp-rs/src/internal.rs){:target="\_blank"}.

```rust
// Implementation of the Send side for SCTP.
pub(crate) async fn sctp_sendmsg_internal(
    fd: &AsyncFd<RawFd>,
    to: Option<SocketAddr>,
    data: SendData,
) -> std::io::Result<()> {
    // Safety: All the pointers are valid because they are within the current scope.
    // Also, this is just a wrapper over `libc` call.
    unsafe {
        let _ = fd.writable().await?;

				...

        let mut sendmsg_header = libc::msghdr {
            msg_name: to_buffer,
            msg_namelen: to_buffer_len,
            msg_iov: &mut send_iov,
            msg_iovlen: 1,
            msg_control,
            msg_controllen,
            msg_flags: 0,
        };

        let cmsg_hdr = libc::CMSG_FIRSTHDR(&sendmsg_header);
        if !cmsg_hdr.is_null() {
            (*cmsg_hdr).cmsg_level = libc::IPPROTO_SCTP;
            (*cmsg_hdr).cmsg_type = CmsgType::SndInfo as i32;
            (*cmsg_hdr).cmsg_len =
                libc::CMSG_LEN(std::mem::size_of::<SendInfo>().try_into().unwrap())
                    .try_into()
                    .unwrap();

            let snd_info = data.snd_info.unwrap();
            std::ptr::copy(
                std::ptr::addr_of!(snd_info) as *const _,
                libc::CMSG_DATA(cmsg_hdr),
                std::mem::size_of::<SendInfo>(),
            );
        }

        let rawfd = *fd.get_ref();

        let flags = 0 as libc::c_int;

				// The following line would result in `EINVAL`.
        let result = libc::sendmsg(rawfd, &mut sendmsg_header as *mut libc::msghdr, flags);
        if result < 0 {
            Err(std::io::Error::last_os_error())
        } else {
            Ok(())
        }
    }
}

```
Since adding `debug` and or `trace` logging was not helping, I had to look for other options for finding out what was the real problem. Since the `libc::sendmsg` would directly call the `sendmsg` system call, one of the first things I tried to look at was - see what was causing the `EINVAL` and reason about it by looking at Kernel's SCTP code. The `sendmsg` `libc` function would call the protocol's (`struct proto` in Linux kernel) `sendmsg` function. In the case of `SCTP`, this is `sctp_sendmsg` inside (`net/sctp/socket.c`). This function returns `EINVAL` from a few places (through called functions), so simply looking at the source code and trying to reason about it was not very straight forward. What was needed was a way to inspect the passed parameters and return values of different functions in the call stack. To be able to do so, I followed the following approach.

# Troubleshooting Pass 1: `pr_debug`

Most modern Linux kernels support what is called as dynamic debugging of the running kernel. At a few places in the source code, there are these `pr_debug` calls, which are similar to `printk` calls, except they are dynamically enabled. [This documentation](https://www.kernel.org/doc/html/v4.11/admin-guide/dynamic-debug-howto.html){:target="\_blank"} provides a good reference for understanding and enabling dynamic debugging in the running Linux Kernel. While trying to enable `pr_debug`, I faced the problem of not being able to initialize the dynamic debugging. This is because, on UEFI secure booted kernels, this functionality cannot be used (due to `kernel_lockdown`). I had to disable secure boot in the BIOS to enable `pr_debug` functionality. [This answer](https://askubuntu.com/questions/1169659/dynamic-debug-permission-issue-k5-0-0){:target="\_blank"} on Ask Ubuntu gives few more details about the specific problem was faced. However, the debug information was quite verbose and there were not enough debug prints (`pr_debug`s) around the function that was likely giving an error, so this did not work out quite well as I would have liked.

Thankfully `sctp` module was built as a loadable kernel module, hence it was possible to rebuild this module with more `pr_debug` and then loading this module will help. Next, I considered re-building this module first to be able to add more debug print information.

# Troubleshooting Pass 2: Re-compiling the module

Compiling kernel modules is something I have not done in the recent past (most likely something like 3-4 years at-least). On modern Ubuntu system, it looks like this is quite involved. Since I did not want to recompile a totally new kernel and also build the modules for that, I was looking at ways where I can compile modules for the current running kernel on my Ubuntu 22.04 (`6.2.0-36-generic`). For this I followed detailed instructions [on this page](https://askubuntu.com/questions/515407/how-recipe-to-build-only-one-kernel-module){:target="\_blank"} to compile the modules, and also enabled `dynamic_debug` in the `/etc/modprobe.d`, so that this can work acrosss reboots.  However, `modprobe` of the newly built module was not working. (I had to actually take a short-cut to only build the module that I am interested in and not build *all* modules from the current kernel config, because building some `kvm` module was giving an error and troubleshooting that was totally out of scope :-) ). Somehow I was not able to get this process right and was not able to do a `modprobe` of my compiled module in the running kernel and re-compiling a stock kernel (and modules) was something I decided against. Then I remembered,about using `uprobe` or `kprobe` it's possible to trace function calls (`uprobe` is for tracing user-space library calls and `kprobe` is for tracing kernel function calls). May be this is a good idea, so the next attempt was trying to debug (and get what actual data is getting sent to the kernel and try to reason out if that makes sense).

# Troubleshooting Part 3: `bpftrace`

[IOvisor](https://www.iovisor.org/){:target="\_blank"} project's [`bpftrace`](https://github.com/iovisor/bpftrace){:target="\_blank"} is a small scripting language used for tracing running programs. This provides scripting around a set of trace points both in the user's space and kernel space. [`bpftrace` tutorial](https://github.com/iovisor/bpftrace/blob/master/docs/tutorial_one_liners.md){:target="\_blank"} is a good source for getting started if you are not aware of what is `bpftrace` and how to use it.

To troubleshoot the original problem, what I decided to do was - add tracepoints at entry and exit of certain kernel functions in the Linux kernel and using the scripting capabilities provided, inspect the passed parameters and return values from those functions. There are two types of probes used `kprobe:<function>` and `kretprobe:<function>` and then the passed arguments and return values can be inspected using the built-in `arg0`,...`argN` and `retval` variables of the `bpftrace` scripting language respectively.

The exact workflow is as follows - write a simple `bpftrace` script and run it using the `bpftrace` utility in one window and run `cargo test` and `cargo test --release` in another window. Look at the information printed in the `bpftrace` window for the successful and failing test cases to isolate the problem.

The script looks like the following. Most parts are quite self explanatory. We are examining the arguments passed to `sctp_sendmsg`.

```bpftrace
probe:sctp_sendmsg {
        printf("arg1_p: %p, arg1: %x, arg2: %x\n", arg1, ((struct sockaddr *)((struct msghdr *)arg1)->msg_name)->sa_family, arg2)
}


kprobe:sctp_sendmsg_check_sflags {
        printf("flags: %d\n", arg1)
}

kprobe:sctp_sendmsg_parse {
        printf("arg3: %p\n", arg3)
}

kretprobe:sctp_sendmsg_parse {
        printf("retval %d\n", retval)
}

kretprobe:sctp_sendmsg {
        printf("sctp_sendmsg retval: %d\n", retval)
}

kretprobe:sctp_sendmsg_to_asoc {
        printf("sctp_sendmsg_to_asoc retval: %d\n", retval)
}

```
Linux kernel's `sctp_sendmsg` looks like following -

```c
static int sctp_sendmsg(struct sock *sk, struct msghdr *msg, size_t msg_len)
{
    struct sctp_endpoint *ep = sctp_sk(sk)->ep;
    struct sctp_transport *transport = NULL;
    struct sctp_sndrcvinfo _sinfo, *sinfo;
    struct sctp_association *asoc, *tmp;
    struct sctp_cmsgs cmsgs;
    union sctp_addr *daddr;
    bool new = false;
    __u16 sflags;
    int err;

    /* Parse and get snd_info */
    err = sctp_sendmsg_parse(sk, &cmsgs, &_sinfo, msg, msg_len);
    if (err)
        goto out;

    sinfo  = &_sinfo;
    sflags = sinfo->sinfo_flags;

    /* Get daddr from msg */
    daddr = sctp_sendmsg_get_daddr(sk, msg, &cmsgs);
    if (IS_ERR(daddr)) {
        err = PTR_ERR(daddr);
        goto out;
    }
		...

out_unlock:
    release_sock(sk);
out:
    return sctp_error(sk, msg->msg_flags, err);
}

```
What was observed was `sctp_sendmsg_parse` didn't fail even for the failed test cases, however the function `sctp_sendmsg_check_sflags` was not getting called, when the test case was failing (or when `sctp_sendmsg` was returning `EINVAL`). A possible problem area was the call to `sctp_sendmsg_get_daddr` was giving an error. Also, we observed that the `sa_family` (in `kprobe:sctp_sendmsg`) was not printing the expected `2` (for `AF_INET`).

Based on the information obtained using `bpftrace` above, we could reason out that there was something wrong with `daddr` above. Also, this issue was coming with `SctpListener`'s `sctp_send` in the Rust crate but not on `ConnectedSocket`'s `sctp_send`, which suggested that in the [API function](https://github.com/gabhijit/ellora/blob/ee2f48de6fdccbc332ea5748f2140f097989651d/sctp-rs/src/internal.rs#L574){:target="\_blank"}, when the `to` parameter was `Some`, there was this problem. The handling of the `Some(to)` parameter is done by the code that looked like following -

```rust

// from file sctp-rs/src/internal.rs
fn sctp_sendmsg_internal(...) {
...
        let (to_buffer, to_buffer_len) = if let Some(addr) = to {
            let os_sockaddr: OsSocketAddr = addr.into();
            (
                os_sockaddr.as_ptr() as *mut libc::c_void,
                os_sockaddr.capacity(),
            )
        } else {
            (std::ptr::null::<OsSocketAddr>() as *mut libc::c_void, 0)
        };

...
}
```
`OsSocketAddr` is provided by [`os_socketaddr`](https://docs.rs/os_socketaddr/latest/os_socketaddr){:target="\_blank"} crate and provides handy utilities for converting Rust's [`SocketAddr`] into `struct sockaddr *` used by a number of `socket` API of `libc`. My initial guess was there is perhaps some alignment issue that is causing the value at the pointer to be correctly read inside the kernel, but looking at the raw pointers above, suggested may be not because the problem occurred regardless of whether the pointer was aligned to 8 bytes or 16 bytes in the `--release` mode but not otherwise. Also, `sendmsg` documentation did not specify anything specific about the alignment requirements of the passed pointer. So perhaps this is not the real cause.

At this point, I was suspecting, may be may be (possibly out of frustration! :-) ), there is some compiler issue this is causing this, that is some optimization is messing around with a pointer? Honestly, this looked a bit far fetched, but when nothing looks like working, one starts suspecting.

But then, after carefully looking at the code, realized that the `to_buffer` above (this is the culprit pointer in the kernel) that was used later in the `sendmsg` call, but by then this `os_sockaddr` structure would have gone out of scope. (Valid only in the `if let ...` scope), and then came the aha! moment. This clearly appears to be the root cause, I was using raw pointer for the structure that has gone out of scope (and hence destroyed), so I am passing a dangling pointer to the kernel and the value is read from that pointer. The actual solution was actually much simple and then it made total sense. All that was really required to be done was making sure that the `os_sockaddr` variable above was still valid when the pointer was accessed. The fixed version of the function is [available here](https://github.com/gabhijit/ellora/blob/ee2f48de6fdccbc332ea5748f2140f097989651d/sctp-rs/src/internal.rs#L574){:target="\_blank"}.

```rust

// from file sctp-rs/src/internal.rs
fn sctp_sendmsg_internal(...) {
...
        // Defining here `os_sockaddr` lasts for the rest of the function call
        let os_sockaddr: OsSocketAddr;
        let (to_buffer, to_buffer_len) = if let Some(addr) = to {
            os_sockaddr = addr.into();
            (
                os_sockaddr.as_ptr() as *mut libc::c_void,
                os_sockaddr.capacity(),
            )
	...
}
```

In my defense though, had the same issue existed both in the `debug` and `release` versions, I'd have been able to troubleshoot this problem earlier perhaps.

# Closing Remarks

One appreciates how an *Undefined Behavior* bites you when one actually faces it. This particular issue would have been trivially caught by the compiler in the *safe* Rust code, by saying the `os_socketaddr` is not in scope when it's reference was accessed. Sometimes we don't quite appreciate the bugs that are avoided in the first place, until we have to deal with *UB*.

Also, a note to self, when looking at any `unsafe` code in future, it might be a good idea to review where the `raw` pointers are accessed and where they come from and make sure the values the pointers are pointing to are still valid when those pointers are going to be `deref`ed, since compiler is not going to come to rescue in `unsafe` blocks. This is documented at enough places, but till one faces the issues, it's not appreciated! :-)

Further, in the future, whenever I have to troubleshoot a problem in the live running code, `bpftrace` would be the first thing to try out before going the path of 're-compile with more debug statements'.

And finally, may be I should consider more unit testing of these `unsafe` parts to identify any more potential issues that may be lurking around undetected. May be I should consider `fuzz` testing as well.

