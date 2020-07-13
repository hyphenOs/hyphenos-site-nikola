.. title: Device Auth and Secure Communication - Implementation on AWS
.. slug: device-secure-communication-implementation
.. date: 2020-07-22 15:32:21 UTC+05:30
.. tags: Systems, Public Cloud, AWS
.. category:
.. link:
.. status:
.. description:
.. author: Abhijit Gadgil
.. type: text
.. summary: The previous post looked at the protocol for the devices that are acting as a client in a REST based device management system. This post discusses details of actually implementing this protocol design on a public cloud.

# Introduction

In order to implement the design presented in the [previous post](/blog/2020/device-secure-communication-design/) on public clouds, a few details are needed to be sorted out. Specifically -

1. When the server side implementation is required to serve a few hundreds to a few thousand devices, high availability is an important requirement. In a cloud environment how is high availability going to be achieved.
2. Related to above requirement, How is the design going to work behind some kind of a _load balancer_ provided by the cloud provider.
3. How does the 'application' receive the client information (_ie_ MAC Address) during each request in order to authenticate the client.

Let's look at these implementation details using AWS Public cloud. While the current implementation discusses AWS, the implementation principles can be applied on any public cloud that provides the required features (_eg_ High Availability through multiple availability zones, Load Balancer etc.). Also, note that the current implementation is a single region (multiple availability zone) implementation.

# Implementation Overview

The following diagram details implementation overview. Main components are enumerated below.

![Security  Implementation with HA ](/blog/images/aws-deployment.png "Security Deployment")

1. The public IP addresses on the load balancer (`ElasticIP-\*`) are the API end points that are resolved through the DNS. (Note: There is no requirement to use cloud provider's DNS, as long as end-points resolve to the public IP addresses on the load balancer, any DNS provider can be used.)

2. At-least two public end-points are available in separate availability zones. In the case of AWS, we are using [Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/index.html) (`NLB-AZ-*`) and not the [Application Load Balancer (`ALB`)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/index.html). The reasons behind this choice are discussed below.

3. We are using EC2 instances (`EC2-AZ-*`) deployed in three availability zones for achieving the high availability in an active-active load balancing. Three instances are required for the consensus protocol of some of the application components (`redis-sentinel`).

4. Each instance is a standalone instance and runs `nginx` based reverse proxy (`R-Proxy-*`) in front of an application server (`App-Server-*`). Both, reverse proxy, application servers and Redis based queue are hosted on the same EC2 instance. All the EC2 instances configuration is identical and thus they can easily be placed in auto-scaling groups. We will be not looking at the internal details of each EC2 instance, beyond the reverse proxy part.

## Summary of AWS Services Used

To provide as a ready reference here is a summary of AWS Services that we have used -

1. AWS Elastic Load Balancer - Network Load Balancing
2. AWS Elastic Compute EC2
3. AWS Elastic File System
4. AWS Relational Database Service RDS (Optional - Application Database)

# Network Load Balancer

As discussed in the design, a client is uniquely identified through Common Name present in the Client Side Certificate. AWS Application Load Balancer (ALB) did not provide this information through `X-HTTP` headers. (Based on a cursory reading it is possible to have `X-HTTP` headers for client IP addresses in AWS ALB, so if that is what one needs, it might be worth looking at.) Thus it was not possible to use ALB. Also, since we were already using Let's Encrypt based server side certificates, it was preferred that the TLS connections are terminated on the `nginx` reverse proxy. Thus in the given implementation, TCP connections are terminated on the AWS Network Load Balancer (NLB) and TLS connections are terminated on the `nginx` reverse proxy.

Some of the configuration options that are selected -

1. NLB is configured for [cross zone load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html#cross-zone-load-balancing). This is required since the high availability instances are deployed in multiple availability zones, we need an NLB to be able to reach instances in other availability zones as well.
2. 'Target Groups' are defined for both **HTTP and HTTPS ports** on all of the backend servers. Note: here the Target groups are defined for `TCP`, the ports used are standard `HTTP`(80) and `HTTPS`(443).These ports map one to one with the corresponding listeners on the load balancer.
3. NLB determines the reachability of the backend servers through health checks. It is possible to have `HTTP` based health checks even for the `TCP` based Target Groups (as defined above), this is actually an interesting feature. The advantage of using this is, with proper configuration in reverse proxy, we can make sure that our 'application' is indeed reachable and not just the reverse-proxy (through Target Groups), even though the Target Groups are TCP based. The URL for our application level health checks is then configured in NLB Target Group configuration and routed via reverse proxy configuration (By using a separate `location` [directive](http://nginx.org/en/docs/http/ngx_http_core_module.html#location).

# Reverse Proxy (nginx)

We are using `nginx` based reverse proxy that is frontending application servers. Most of the heavy lifting of the client side certificates design is performed by `nginx`. The reverse proxy, terminates TLS sessions, thus it has access to the client's TLS certificate. The information from the client's TLS certificate is extracted using the [nginx SSL variables](https://nginx.org/en/docs/http/ngx_http_ssl_module.html). Using this information a suitable `X-HTTP-Header` is added to _every_ request using `nginx`'s `proxy_set_header` [directive](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header). So the relevant configuration for obtaining data from Client Side Certificates looks like below (other details omitted) -

```shell

server {

	....

  # Other SSL specific config etc.
	ssl_client_certificate /path/to/ca-chain-cert;
	ssl_verify_client required; # set based on your needs to `optional`


  ...
	proxy_set_header X-Client-Info $ssl_client_s_dn;

}
```

Additionally, in a high availability configuration, the SSL/TLS specific configuration (server's key and certificate, client's CA certificate chain) is required to be available to all the instances of reverse proxy that can be potentially running in different AZs. The simplest mechanism to achieve this is keeping this configuration on an AWS Elastic File Service (`EFS`) which can then be read-only mounted by each of the instances running reverse proxy (and application servers). This setting can be templatized through `fstab` configuration.

# Application Servers

Application server will leverage the HTTP header information in the request to authenticate the clients. An overall idea of this is explained with a code that looks like below. This is more like a pseudo-code but that explains the idea. Note: Some details related to device state and type of the message are omitted below, but this should give an idea about how one can access the common name in the application server that is behind a load balancer and a reverse proxy.

```python

mac_id = request.META.get('HTTP_X_CLIENT_INFO')
if mac_id is None:
    return 403

if mac_id in provisioned_devices:
    if mac_id == device.mac_id:
        return 200
    else:
        return 403
else:
    return 404

  ...

```


# Life of a Request

After having looked at each of the main components in the implementation, we will now trace the life of a single request as it goes through this implementation. This should be quite useful in understanding how the services discussed above fit together.

In this section we capture all the discussion in the previous sections to understand what happens to a given request in the configuration above and how this can achieve our security protocol design.

1. Client (the device that is managed) obtains the IP address of the server through the DNS. This IP address is one of the Elastic IPs that are exposed by the NLB.

2. Client completes a TCP connection with the NLB. NLB chooses one of the backend servers that is 'healthy' (as determined by health check for the target group - in our case port 443 target group.)

3. The NLB initiates a TCP connection on the port 443 of the chosen 'healthy' instance. This connection is then terminated by `nginx`.

4. The `TLS` handshake is then completed between the client and `nginx` obtains the client certificate and sets up the `X-Client-Info` HTTP header.

5. The request is then forwarded to application server.

6. Application server obtains the client information from the HTTP header and authenticates the client against provisioned devices.

7. Once the client can be authenticated, the client's state machine discussed in the protocol in the [previous post](/blog/2020/device-secure-communication-design/) comes into play for authentication of client messages.

# Wrapping Up

To implement an application with slightly non-standard requirements (viz. Client Side Certificate Support, Custom Server Certificates, High Availability etc.), requires to carefully consider the features provided by individual 'cloud' services and sometimes work around or work with those services. One detailed that we have omitted right now as a part of maintaining this design is how do we periodically update Let's Encrypt certificates, which are available only for 90 days and hence need to be periodically refreshed. There are certain quirks specific to way Let's Encrypt handles obtaining certificates. That we'll discuss in a separate post later.

Also, we have discussed which configuration options to choose, we have not discussed in substantial details actual configurations. Feel free to add comments and I'll try to answer those details.
