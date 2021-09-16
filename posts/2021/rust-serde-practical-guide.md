.. title: Rust `serde` Practical Guide
.. slug: rust-serde-practical-guide
.. date: 2021-09-16 10:02:00 UTC+05:30
.. tags: Rust, Performance, FlameGraphs
.. author: Abhijit Gadgil
.. link:
.. description:
.. type:
.. status: draft
.. summary: This blog post explores some practical tips while using with the excellent `serde` crate.

# Introduction

Often it is necessary to serialize or de-serialize a user defined structure into some format on wire or on disk. For example, serializing a `struct` to a JSON while working with REST APIs. Rust's `serde` crate provides functionality to make this quite straight forward. Also, `serde` crate provides plenty of customization options during serializing or de-serializing. In this post we explore some concepts as well as examples of custom serialization / de-serialization tips while using `serde`.

While using `serde`, simply deriving the `Serialize` and `DeSerialize` on a user defined structure is usually good enough. Things start to get little challenging, when one has to customize the serialize/deserialize behavior. While `serde` provides a number of [attributes](https://serde.rs/attributes.html){:target="\_blank"} to make this easier, there are still about 5 percent cases where one might be required to do little more than simply using those `attributes`. To a first time user of `serde`, it can be quite daunting especially when one reads example code that has  very similar looking `struct`s or `trait`s like `Serialize`, `Serializer` _etc_ and one is not quite sure what they should do to get started. In this blog we will primarily be looking at the cases where the user has to perform such customizations along with some other practical tips.

# Some Concepts

In `serde` there are primarily two concepts, [data structures and data formats](https://serde.rs/#serde){:target=\_blank}.

A 'Data Structure' can be anything that implements a `Serialize` and/or `DeSerialize` trait. Note: It's not at all necessary that one has to implement both `Serialize` / `DeSerialize` traits for their custom data structure. Typically, if you are developing some APIs, likely you will be required to implement both of these traits as you will be required to both accept data from the other end of the API and as well as send data to the other end of the API. However if you are simply implementing a `Reader` for something, simply implementing `DeSerialize` may be sufficient. If one just wants to dump a structure into JSON say, then simply implementing `Serialize` is sufficient.

A 'Data Format' is something that represents a structure on wire (or on disk) and it needs to know how individual fields in a `struct` map to their on wire representation. In `serde` speak an example of a Data Format is JSON. Typically, third party crates ( _eg_ [`serde_json`](){:target="\_blank"}) will implement sensible defaults for the serialization and de-serialization part for a given data format, this they will typically do by implementing a `Serializer` and/or `Deserializer` trait.

Thus, an important thing to remember is - if you are implementing a fancy new data format for representing things on wire, you will typically be implementing the `Serializer`/`Deserializer` trait (note: this is something one is rarely required to do), but normally to simply be able to `Serialize`/`Deserialize` custom data structures simply `derive`ing those two traits is enough and `serde` will do all the heavy lifting for you. Very often there are requirements like you want to do simple modifications to the representation of a field of a `struct` for example converting the `struct` `field` name to a `CamelCase` etc or renaming the fields when represented on wire (this is especially true for field names that can be Rust keywords.)

# Custom Serialization/DeSerialization

To help understand the concepts described in the previous section, let's look at some examples, to understand how this can be achieved. `serde`'s documentation is quite detailed for the built-in customization. Such a customization is possible over a container type (like a `struct` or an `enum`) or a variant of an `enum` or individual field of a `struct`. All available options are described [here](){:target="\_blank}). But it's possible to even go beyond what is provided by default.

## Custom Serialization Example (integers as `hex`)

Let's say we have a field on a `struct` that is some integer type (say `u16`), the default serialization behavior for integer types in `serde` is the decimal representation of the integer. However if we would like to serialize this field to it's hex representation, there are a couple of ways this can be done. One will be to define a wrapping `struct` type on the integer type and then implement a custom serializer/de-serializer for it that looks something like below (Please note we are not using a `#[derive(Serialize)]` on the `struct` below.

In the following discussions we will be talking a lot about `JSON` data format, but the discussion remains just as much applicable for other data formats, JSON being very common, most of our examples focus on that.

```rust
struct U16(u16);

impl Serialize for U16 {
	fn serialize(&serializer: S) -> Result<S::Ok, S::Error>
	where
		S: Serializer,
	{
		...
	}
}

```
However, defining a `struct` just for this alone is often too much of boiler-plate code, instead we can use a `field` `attribute` on the `struct` and simply derive `Serialize` on rest of the `struct`.

```rust

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
This is much lesser boilerplate code achieving the same result. But former option may be useful if one has to implement other traits as well. Note here, while we are implementing the `trait Serialize` on the `struct`, the `trait` in the `where` clause is a `Serializer`. Also, note the method `serialize_str`. The `Serializer` trait defines methods for all the `Rust` primitive and user defined types. This trait will typically be implemented by the `Data Format` as we described above.

## Custom De-Serialization example (Default when JSON `null`)

In many APIs there might be some fields which are optional or if present null and in such cases, we may want to use the default value of the field type if it exists (_ie_ if the field type implements `Default` trait). This can be done by again implementing a custom 'de-serialization' function [as follows](){:target="\_blank})

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

Similar to example above, note the `trait` bounds for the types `T` and `D`. `T` in this case is the Data Structure (which implements `Deserialize` trait, but in this example it also has to implement a `Default` trait, since we need `default` value), `D` is the Data Format that implements `Deserializer` trait.


## Custom Serialization of a `struct` implementing a Trait

Example of custom deserialization
