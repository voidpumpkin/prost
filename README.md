![continuous integration](https://github.com/tokio-rs/prost/workflows/continuous%20integration/badge.svg)
[![Documentation](https://docs.rs/prost/badge.svg)](https://docs.rs/prost/)
[![Crate](https://img.shields.io/crates/v/prost.svg)](https://crates.io/crates/prost)
[![Dependency Status](https://deps.rs/repo/github/tokio-rs/prost/status.svg)](https://deps.rs/repo/github/tokio-rs/prost)
[![Discord](https://img.shields.io/discord/500028886025895936)](https://discord.gg/tokio)

# PROSIT

`prosit` is a [Protocol Buffers](https://developers.google.com/protocol-buffers/)
implementation for the [Rust Language](https://www.rust-lang.org/), based on [`prost`](https://github.com/danburkert/prost).
`prosit` extends `prost` with different code generation options, adding more idiomatic Rust types according to the [`parse, don't validate`](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) philosophy.

To add prosit to your project as an alternative code generator, use the patch feature of cargo, and use prost inside your codebase.

```ignore
[dependencies]
prost = "0.11"
# Only necessary if using Protobuf well-known types:
prost-types = "0.11"
```

Once prosit exits experimental stage, it will be published on Cargo, but hopefully as an extension to `prost`

See the [snazzy repository](https://github.com/danburkert/snazzy) for a simple
start-to-finish example.

### MSRV

`prost` follows the `tokio-rs` projects MSRV model and supports 1.60. For more
information on the tokio msrv policy you can check it out [here][tokio msrv]

[tokio msrv]: https://github.com/tokio-rs/tokio/#supported-rust-versions

## Generated Code

`prost` generates Rust code from source `.proto` files using the `proto2` or
`proto3` syntax. `prost`'s goal is to make the generated code as simple as
possible.

### `protoc`

With `prost-build` v0.11 release, `protoc` will be required to invoke
`compile_protos` (unless `skip_protoc` is enabled). Prost will no longer provide
bundled a `protoc` or attempt to compile `protoc` for users. For install
instructions for `protoc` please check out the [protobuf install] instructions.

[protobuf install]: https://github.com/protocolbuffers/protobuf#protocol-compiler-installation

### Packages

Prost can now generate code for `.proto` files that don't have a package spec.
`prost` will translate the Protobuf package into
a Rust module. For example, given the `package` specifier:

[package]: https://developers.google.com/protocol-buffers/docs/proto#packages

```protobuf,ignore
package foo.bar;
```

All Rust types generated from the file will be in the `foo::bar` module.

### Messages

Given a simple message declaration:

```protobuf,ignore
// Sample message.
message Foo {
}
```

`prost` will generate the following Rust struct:

```rust,ignore
/// Sample message.
#[derive(Clone, Debug, PartialEq, Message)]
pub struct Foo {
}
```

### Fields

Fields in Protobuf messages are translated into Rust as public struct fields of the
corresponding type.

#### Scalar Values

Scalar value types are converted as follows:

| Protobuf Type | Rust Type |
| ------------- | --------- |
| `double`      | `f64`     |
| `float`       | `f32`     |
| `int32`       | `i32`     |
| `int64`       | `i64`     |
| `uint32`      | `u32`     |
| `uint64`      | `u64`     |
| `sint32`      | `i32`     |
| `sint64`      | `i64`     |
| `fixed32`     | `u32`     |
| `fixed64`     | `u64`     |
| `sfixed32`    | `i32`     |
| `sfixed64`    | `i64`     |
| `bool`        | `bool`    |
| `string`      | `String`  |
| `bytes`       | `Vec<u8>` |

#### Enumerations

All `.proto` enumeration types convert to the Rust `i32` type. Additionally,
each enumeration type gets a corresponding Rust `enum` type. For example, this
`proto` enum:

```protobuf,ignore
enum PhoneType {
  MOBILE = 0;
  HOME = 1;
  WORK = 2;
}
```

gets this corresponding Rust enum [^1]:

```rust,ignore
pub enum PhoneType {
    Mobile = 0,
    Home = 1,
    Work = 2,
}
```

[^1]: Annotations have been elided for clarity. See below for a full example.

You can convert a `PhoneType` value to an `i32` by doing:

```rust,ignore
PhoneType::Mobile as i32
```

The `#[derive(::prost::Enumeration)]` annotation added to the generated
`PhoneType` adds these associated functions to the type:

```rust,ignore
impl PhoneType {
    pub fn is_valid(value: i32) -> bool { ... }
    pub fn from_i32(value: i32) -> Option<PhoneType> { ... }
}
```

so you can convert an `i32` to its corresponding `PhoneType` value by doing,
for example:

```rust,ignore
let phone_type = 2i32;

match PhoneType::from_i32(phone_type) {
    Some(PhoneType::Mobile) => ...,
    Some(PhoneType::Home) => ...,
    Some(PhoneType::Work) => ...,
    None => ...,
}
```

Additionally, wherever a `proto` enum is used as a field in a `Message`, the
message will have 'accessor' methods to get/set the value of the field as the
Rust enum type. For instance, this proto `PhoneNumber` message that has a field
named `type` of type `PhoneType`:

```protobuf,ignore
message PhoneNumber {
  string number = 1;
  PhoneType type = 2;
}
```

will become the following Rust type [^2] with methods `type` and `set_type`:

```rust,ignore
pub struct PhoneNumber {
    pub number: String,
    pub r#type: i32, // the `r#` is needed because `type` is a Rust keyword
}

impl PhoneNumber {
    pub fn r#type(&self) -> PhoneType { ... }
    pub fn set_type(&mut self, value: PhoneType) { ... }
}
```

Note that the getter methods will return the Rust enum's default value if the
field has an invalid `i32` value.

The `enum` type isn't used directly as a field, because the Protobuf spec
mandates that enumerations values are 'open', and decoding unrecognized
enumeration values must be possible.

[^2]: Annotations have been elided for clarity. See below for a full example.

#### Field Modifiers

Protobuf scalar value and enumeration message fields can have a modifier
depending on the Protobuf version. Modifiers change the corresponding type of
the Rust field:

| `.proto` Version  | Modifier   | Rust Type                                   |
| ----------------- | ---------- | ------------------------------------------- |
| `proto2`          | `optional` | `Option<T>`                                 |
| `proto2`          | `required` | `T`                                         |
| `proto3`          | default    | `T` for scalar types, `Option<T>` otherwise |
| `proto3`          | `optional` | `Option<T>`                                 |
| `proto2`/`proto3` | `repeated` | `Vec<T>`                                    |

Note that in `proto3` the default representation for all user-defined message
types is `Option<T>`, and for scalar types just `T` (during decoding, a missing
value is populated by `T::default()`). If you need a witness of the presence of
a scalar type `T`, use the `optional` modifier to enforce an `Option<T>`
representation in the generated Rust struct.

#### Map Fields

Map fields are converted to a Rust `HashMap` with key and value type converted
from the Protobuf key and value types.

#### Message Fields

Message fields are converted to the corresponding struct type. The table of
field modifiers above applies to message fields, except that `proto3` message
fields without a modifier (the default) will be wrapped in an `Option`.
Typically message fields are unboxed. `prost` will automatically box a message
field if the field type and the parent type are recursively nested in order to
avoid an infinite sized struct.

#### Oneof Fields

Oneof fields convert to a Rust enum. Protobuf `oneof`s types are not named, so
`prost` uses the name of the `oneof` field for the resulting Rust enum, and
defines the enum in a module under the struct. For example, a `proto3` message
such as:

```protobuf,ignore
message Foo {
  oneof widget {
    int32 quux = 1;
    string bar = 2;
  }
}
```

generates the following Rust[^3]:

```rust,ignore
pub struct Foo {
    pub widget: Option<foo::Widget>,
}
pub mod foo {
    pub enum Widget {
        Quux(i32),
        Bar(String),
    }
}
```

`oneof` fields are always wrapped in an `Option`.

[^3]: Annotations have been elided for clarity. See below for a full example.

### Services

`prost-build` allows a custom code-generator to be used for processing `service`
definitions. This can be used to output Rust traits according to an
application's specific needs.

### Generated Code Example

Example `.proto` file:

```protobuf,ignore
syntax = "proto3";

package proto.api.v1;

import "api/rust.proto";

message Data {
  // generates the Uuid type instead of the string type
  string id = 1 [(rust.codegen).type = UUID];

  // does not wrap Foo in an option, but instead errors during parsing if foo is missing.
  Foo foo = 2 [(rust.codegen).required = true];

message Foo {
  // either one of the oneof messages has to be present in the message. Protobuf does not define attributes
  // on oneoffs, so this is message level.
  option (rust.message).oneofs_required = true;

  oneof biz {
    option (rust.required) = true;

    string bar = 1;
    string bax = 2;
  }
}
```

This wil generate approximately the following code:

```rust,ignore
#[derive(Clone, PartialEq, ::prost::Message)]
pub struct Person {
    #[prost(string, tag="1")]
    pub name: ::prost::alloc::string::String,
    /// Unique ID number for this person.
    #[prost(int32, tag="2")]
    pub id: i32,
    #[prost(string, tag="3")]
    pub email: ::prost::alloc::string::String,
    #[prost(message, repeated, tag="4")]
    pub phones: ::prost::alloc::vec::Vec<person::PhoneNumber>,
}

pub struct Foo {
  biz: Biz,
}

pub enum Biz {
  Bar(String),
  Baz(String),
}
```

## Accessing the `protoc` `FileDescriptorSet`

The `prost_build::Config::file_descriptor_set_path` option can be used to emit a file descriptor set
during the build & code generation step. When used in conjunction with the `std::include_bytes`
macro and the `prost_types::FileDescriptorSet` type, applications and libraries using Prost can
implement introspection capabilities requiring details from the original `.proto` files.

## Using `prost` in a `no_std` Crate

`prost` is compatible with `no_std` crates. To enable `no_std` support, disable
the `std` features in `prost` and `prost-types`:

```ignore
[dependencies]
prost = { version = "0.6", default-features = false, features = ["prost-derive"] }
# Only necessary if using Protobuf well-known types:
prost-types = { version = "0.6", default-features = false }
```

Additionally, configure `prost-build` to output `BTreeMap`s instead of `HashMap`s
for all Protobuf `map` fields in your `build.rs`:

```rust,ignore
let mut config = prost_build::Config::new();
config.btree_map(&["."]);
```

When using edition 2015, it may be necessary to add an `extern crate core;`
directive to the crate which includes `prost`-generated code.

## Serializing Existing Types

`prost` uses a custom derive macro to handle encoding and decoding types, which
means that if your existing Rust type is compatible with Protobuf types, you can
serialize and deserialize it by adding the appropriate derive and field
annotations.

Currently the best documentation on adding annotations is to look at the
generated code examples above.

### Tag Inference for Existing Types

Prost automatically infers tags for the struct.

Fields are tagged sequentially in the order they
are specified, starting with `1`.

You may skip tags which have been reserved, or where there are gaps between
sequentially occurring tag values by specifying the tag number to skip to with
the `tag` attribute on the first field after the gap. The following fields will
be tagged sequentially starting from the next number.

```rust,ignore
use prost;
use prost::{Enumeration, Message};

#[derive(Clone, PartialEq, Message)]
struct Person {
    #[prost(string, tag = "1")]
    pub id: String, // tag=1
    // NOTE: Old "name" field has been removed
    // pub name: String, // tag=2 (Removed)
    #[prost(string, tag = "6")]
    pub given_name: String, // tag=6
    #[prost(string)]
    pub family_name: String, // tag=7
    #[prost(string)]
    pub formatted_name: String, // tag=8
    #[prost(uint32, tag = "3")]
    pub age: u32, // tag=3
    #[prost(uint32)]
    pub height: u32, // tag=4
    #[prost(enumeration = "Gender")]
    pub gender: i32, // tag=5
    // NOTE: Skip to less commonly occurring fields
    #[prost(string, tag = "16")]
    pub name_prefix: String, // tag=16  (eg. mr/mrs/ms)
    #[prost(string)]
    pub name_suffix: String, // tag=17  (eg. jr/esq)
    #[prost(string)]
    pub maiden_name: String, // tag=18
}

#[derive(Clone, Copy, Debug, PartialEq, Eq, Enumeration)]
pub enum Gender {
    Unknown = 0,
    Female = 1,
    Male = 2,
}
```

## Nix

The prost project maintains flakes support for local development. Once you have
nix and nix flakes setup you can just run `nix develop` to get a shell
configured with the required dependencies to compile the whole project.

## FAQ

1. **Could `prost` be implemented as a serializer for [Serde](https://serde.rs/)?**

Probably not, however I would like to hear from a Serde expert on the matter.
There are two complications with trying to serialize Protobuf messages with
Serde:

- Protobuf fields require a numbered tag, and currently there appears to be no
  mechanism suitable for this in `serde`.
- The mapping of Protobuf type to Rust type is not 1-to-1. As a result,
  trait-based approaches to dispatching don't work very well. Example: six
  different Protobuf field types correspond to a Rust `Vec<i32>`: `repeated
int32`, `repeated sint32`, `repeated sfixed32`, and their packed
  counterparts.

But it is possible to place `serde` derive tags onto the generated types, so
the same structure can support both `prost` and `Serde`.

2. **I get errors when trying to run `cargo test` on MacOS**

If the errors are about missing `autoreconf` or similar, you can probably fix
them by running

```ignore
brew install automake
brew install libtool
```

## License

`prost` is distributed under the terms of the Apache License (Version 2.0).

See [LICENSE](https://github.com/tokio-rs/prost/blob/master/LICENSE) for details.

Copyright 2022 Dan Burkert & Tokio Contributors
