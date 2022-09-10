.. title: Rust `serde` Practical Guide
.. slug: rust-serde-practical-guide
.. date: 2022-10-04 18:02:00 UTC+05:30
.. tags: Rust, Serde, Serialization, De-serialization
.. author: Abhijit Gadgil
.. link:
.. description:
.. type:
.. status:
.. summary: This blog post explores some practical tips while using with the excellent `serde` crate.

# Introduction

Often it is necessary to serialize or deserialize a user defined structure into some format on wire or on disk. For example, serializing a `struct` to a JSON while working with REST APIs. Rust's `serde` crate provides functionality to make this quite straight forward.  While using `serde`, often, using the `derive` macros `Serialize` and `DeSerialize` on a user defined structure is usually good enough. Things start to get little challenging, when one has to customize the serialize/deserialize behavior. While `serde` provides a number of [attributes](https://serde.rs/attributes.html){:target="\_blank"} to make this easier, there are still some cases where one might be required to do little more than simply using those `attributes`. To a first time user of `serde`, it can be quite daunting especially when one reads example code that has  very similar looking `struct`s or `trait`s like `Serialize`, `Serializer` _etc_ and one is not quite sure what they should do to get started. In this blog we will primarily be looking at the cases where the user has to perform such customizations along with some other practical tips. One goal of this post is to also help reader in understanding the excellent [`serde` documentation](https://serde.rs/){:target=\_blank}, which at-least I have found challenging reading first few times.

# Some Concepts

In `serde` there are many 'traits'/'structs' that have similar looking names and that makes it a bit confusing. First, let's take a look at those -

1. There are two derive macros `Serialize` and `DeSerialize`.
2. There are also traits with same name as `Serialize` and `DeSerialize`.
3. Also there are additional traits `Serializer` and `DeSerializer`.

We'll now see how they are related. In `serde` there are primarily two concepts, [data structures and data formats](https://serde.rs/#serde){:target=\_blank}.

A 'Data Structure' is typically any Rust user defined type. One will typically implement the trait `Serialize` and/or `DeSerialize` for these structures. If one simply wishes to use the sensible defaults that `serde` provides, these traits can be implemented using the `derive` macros above. For most common use cases this is suffice. There are cases, when one has to implement these 'traits' by hand. We will see some examples of that shortly.

A 'Data Format' on the other hand is something that represents a structure on wire (or on disk) and it needs to know how individual fields in a `struct` map to their on wire representation. In `serde` an example of a Data Format is JSON. Typically, third party crates ( _eg_ [`serde_json`](https://docs.rs/serde_json/latest/serde_json/){:target="\_blank"}) will implement sensible defaults for such serialization and de-serialization part for a given data format, this they will typically do by implementing a `Serializer` and/or `Deserializer` trait from the `serde` crate on a `struct` defined in that trait. For example, you will see a [`struct Serializer`](https://github.com/serde-rs/json/blob/master/src/ser.rs#L14){:target="\_blank"} from the crate `serde-json` that implements the [`Serializer`](https://github.com/serde-rs/serde/blob/master/serde/src/ser/mod.rs#L331){:target="\_blank"} trait from the crate `serde`.

Also, note that once `Serialize`/`DeSerialize` trait is implemented on a data structure, it can be used with any data format. The 'Data Structure's and 'Data Format' interact through what is called as [`serde` Data Model](https://serde.rs/data-model.html){:target="\_blank"}

Simplest thing to keep in mind is -

1. If you are just defining new 'struct's that have to be represented on wire using one of the supported 'Data Formats', simply using derive macros is enough.
2. If you need to tweak the implementation of a serializer or a de-serializer, you may need to implement the `Serialize`/`DeSerialize` trait for your structure, but before that see if the attributes provided by serde are sufficient.
3. If you are defining a new on wire or an on disk format, you will need to implement the `Serializer`/`DeSerializer` in your crate on one of the structures from your crate.

# Custom Serialization/Deserialization

To help understand the concepts described in the previous section, let's look at some examples. `serde`'s documentation is quite detailed for the built-in customization. Such a customization is possible over a container type (like a `struct` or an `enum`) or a variant of an `enum` or individual field of a `struct`. All available options are described [here](https://serde.rs/custom-serialization.html){:target="\_blank}). But it's possible to even go beyond what is provided by default. Some [examples](https://serde.rs/examples.html){:target="\_blank} are available that explain some more custom serialization.

In the following discussions we will be talking a lot about `JSON` data format, but the discussion remains just as much applicable for other data formats, JSON being very common, most of our examples focus on that. We will take a couple of more examples of 'custom' serialization.

## Custom Serialization Example (integers as `hex`)

Let's say we have a field in a `struct` that is some integer type (say `u16`), the default serialization behavior for integer types in `serde` is the decimal representation of the integer. However if we would like to serialize this field to it's hex representation, there are a couple of ways this can be done. First, one can define a wrapping `struct` type on the integer type and then implement `Serialize` (or `Deserialize`) for it that looks something like below (Please note we are not using a `#[derive(Serialize)]` on the wrapping `struct` below.


```rust
// No `[#derive(Serialize)] on this struct (or else it will use default decimal).
struct U16(u16);

impl Serialize for U16 {
	fn serialize(&serializer: S) -> Result<S::Ok, S::Error>
	where
		S: Serializer,
	{
		...
	}
}

#[derive(Serialize)]
struct Foo {
	bar: U16,
}

```
However, defining a wrapping `struct` just for this alone is often too much of boilerplate code, instead we can use a `field` `attribute` on the `struct` and simply derive `Serialize` on rest of the `struct`.

```rust

#[derive(Serialize)]
struct Foo {
	#[serde(serialize_with="number_as_hex")]
	bar: u16,
}

fn number_as_hex(value: u16, serializer: S) -> Result<S::Ok, S::Result>
	where
		S: Serializer,
{
	serializer.serialize_str(format!("0x02x", value).as_str())
}
```
This is much lesser boilerplate code achieving the same result. Note here, while we are implementing the `trait Serialize` on the `struct`, the `trait` in the `where` clause is a `Serializer`. Also, the method `serialize_str` is one of the `trait` methods of the `Serializer` trait. The `Serializer` trait defines methods for all the `Rust` primitive and other types like `struct`, `enum` _etc_. This trait will typically be implemented by a crate that is implementing the `Data Format`.

## Custom Deserialization example (Default when JSON `null`)

In many APIs there might be some fields which are optional or if present null and in such cases, we may want to use the default value of the field type if it exists (_ie_ if the field type implements `Default` trait). This can be done by again implementing a custom 'deserialization' function [as follows](https://stackoverflow.com/a/65705111/2845044){:target="\_blank})

```rust
#[derive(Deserialize)]
struct Foo {
   #[serde(deserialize_with = "deserialize_null_default")]
   value: String,
}

fn deserialize_null_default<'de, D, T>(deserializer: D) -> Result<T, D::Error>
where
    T: Default + Deserialize<'de>,
    D: Deserializer<'de>,
{
    let opt = Option::deserialize(deserializer)?;
    Ok(opt.unwrap_or_default())
}
```

Note the `trait` bounds for the types `T` and `D`. `T` in this case is the Data Structure (which implements `Deserialize` trait, but in this example it also has to implement a `Default` trait, since we need `default` value), `D` is the Data Format type that implements `Deserializer` trait.


## Custom Serialization of `Trait` objects

Sometimes, it's possible that a structure that we are serializing has got a field that is a type implementing a trait _eg_ A vector of `Box<dyn CustomTrait>`. It is possible to simply skip serializing that particular field or in some cases it may even be required to be able to `Serialize` (and `Deserialize`) that field. This is a bit involved and `erased_serde` crate provides the functionality and performs the heavy lifting for it. Usually a couple of things are required to be done. The `trait` whose `trait` `objects` are to be serialized will have to be Sub trait of `erased_serde::Serialize`. This is explained in the documentation of [`erased_serde`](https://docs.serde.rs/erased_serde/index.html){:target="\_blank"}.

We will look at a concrete example from one of the projects that we implemented where this feature of `serde` was quite handy. We wanted to 'dump' contents of a Packet after parsing individual layers in the packet as a Json. Each of these layers implemented a `Layer` trait and a packet was a collection of all such `Layer` traits. An additional requirement was that each `Layer` is to be identified by it's Layer name ( A function to obtain that was already a required method in the `Layer` trait.). The following code shows the details of this implementation. We are mostly highlighting the parts that are important. A complete example can be seen in the [repository](https://github.com/gabhijit/scalpel){:target="\_blank"} in [`packet.rs`](https://github.com/gabhijit/scalpel/blob/master/src/packet.rs){:target="\_blank"} and [`layer.rs`](https://github.com/gabhijit/scalpel/blob/master/src/layer.rs){:target="\_blank"}.

```rust
use erased_serde::serialize_trait_object;

pub trait Layer: Debug + erased_serde::Serialize {
	...
}

serialize_trait_object!(Layer);

...

#[derive(Debug, Default, Serialize)]
pub struct Packet<'a> {
		...
    #[serde(serialize_with = "serialize_layers_as_struct")]
    pub layers: Vec<Box<dyn Layer>>,
		...
}

fn serialize_layers_as_struct<S>(
    layers: &Vec<Box<dyn Layer>>,
    serializer: S,
) -> Result<<S as Serializer>::Ok, <S as Serializer>::Error>
where
    S: Serializer,
{
    let mut state = serializer.serialize_struct("layers", layers.len())?;
    for layer in layers {
        state.serialize_field(layer.short_name(), layer)?;
    }
    state.end()
}

```

In the example above even though the field `layers` is a `Vec`, since we want to be able to have the name of each layer available along with the it's serialized value, we are implementing a custom serialize function where the `Vec` is actually serialized as a `struct` using the `Serializer` `trait`'s `serialize_struct` method and individual members of the `Vec` are serialized as if they were `struct` fields. This achieves our desired output.

# Conclusion

In this post we looked at some of the ways Serialization and Deserialization of Rust types can be customized. Some of the important things to remember are -

1. `Serialize` is a trait that is to be implemented by the Data Structure to be serialized.
2. `Serializer` is a trait that is to be implemented by the `struct` (or any type) implementing a Data Format.
3. Typically a Data Format implementor will provide an API that will internally - generate a serializer `struct` (implementing the `Serializer` trait), pass the instance of the serializer `struct` to the `serialize` method of the value (defined in `Serialize` trait) and returns the result (often a `String` or `u8` buffer.)

Most examples in this blog post are taken from the [`scalpel`](https://github.com/gabhijit/scalpel){:target="\_blank"} packet dissection crate.
