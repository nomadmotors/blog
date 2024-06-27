+++
title = 'Embedded Command: Serialization'
date = 2024-06-27T10:43:03-07:00
+++

When dealing with little embedded ecosystems, one of the most important -- and often critical -- design considerations is, "how can information be shared?"

Even for the simplest of embedded systems, communication with a phone, WiFi, or a peripheral over a wire is a common requirement. And with larger systems, information may need to be passed between multiple MCUs in order to conduct the operations of the system as a whole.

One can imagine just how important it is for data to arrive *intact* and *timely* when dealing with safety-critical embedded systems.

This post is the first of a series, **Embedded Command** where we will build a robust, performant, and *safe* communication framework for embedded devices piece by piece.

# How do we represent Data?

The first question to ponder, is how exactly we even want our communication process to look.

We know that in our programs we have **types** which represent the *shape* of our data, but these types only exist as abstract structures for the compiler and developer to use harmoniously to design structure, control flow, and enforcements. The notion of types *does not* exist outside of our program. Since we are dealing with communication between multiple devices, we need to create a unified description of how data should look.

To fascilitate communication, **serial data** is the preferred choice, whether it be UART, BLE, Sockets, etc. So we will design our system around that. The process of converting in-memory data to and from *bytes* -- or whatever the word size is for a serial protocol -- is called **serialization**.

Ok so we want to be able to turn *types* to and from *bytes*, basically. What does that look like? Could we do something like **JSON**?

JSON is what's called "self-describing", which means that you can read JSON data, knowing nothing about the shape of the data you are receiving, and completely construct data in-memory from that. This is cool, and highly dynamic. The [serde](https://serde.rs) crate provides an expansive interface for this.

This does not work, however, for our use-case in an embedded context because of just how dynamic it is. For embedded development, our job is to maximize invariance and static analysis. Serializing to JSON is an intrisicaly dynamic process.

So we probably want a *non self-describing* serialization protocol. Meaning that the sender includes no descriptive information within the serialized data, and the receiver must be expecting data of the appropriate "shape".

> That might sound strange right now, how can the receiver "expect" a certain kind of message to be sent if the message hasn't been sent yet, and we will elaborate on how this works later.

# The Interface

Now that we understand what needs to be done, let's design a Rust interface to fascilitate this. We'll make a sub-crate of `embedded-command` called `cookie-cutter`.

Let's make a trait called *Serialize*, where types that implement this trait can be serialized to and from some kind of transmission data type:

```rust
trait Serialize {
    fn serialize(&self, dst: ?);
    fn deserialize(src: ?) -> Self;
}
```

Ok... this interface has a good start, but what exactly should the function parameters be? And... come to think of it, do we need to know the *size* of the data we are serializing? What would it even look like to serialize an unsized type? How could that process be static?

Let's break this down and tackle each problem.

## Size

The prospect of transmitting sized types is alluring, as the size of the type would be baked into the binary at compile time, increasing the invariance of the serialization process. But on the other hand, what if some types cannot be sized, like strings or vectors?

Well, it really would be a shame for each of the two types of types to bog down each other, so how about we split the interface in two. One for types that have a static size, and one for types that don't!

This way, if the situation does not demand unsized types, we can reap the reward of having a perfectly static communication system.

```rust
trait UnsizedSerialize {
    fn serialize(&self, dst: ?);
    fn deserialize(src: ?) -> Self;
}

trait SizedSerialize {
    fn serialize(&self, dst: ?);
    fn deserialize(src: ?) -> Self;
}
```

The unsized types will probably need to describe themselves a bit (like the number of bytes to expect since that is no longer statically known) but it will still be very low overhead.

## Parameters

Hmm, surely the parameters for these two traits will be different, for `SizedSerialize` the size of the type should be known in some way, so let's add that as an associated const of the trait:

```rust
trait SizedSerialize {
    const SIZE: usize;

    fn serialize(&self, dst: ?);
    fn deserialize(src: ?) -> Self;
}
```

...and while we're at it, now that the implementer must (somehow) provide us with the size of the type in serialized form, we can serialize to and from an array of that size!

```rust
trait SizedSerialize {
    const SIZE: usize;

    fn serialize(&self, dst: &mut [?; Self::SIZE]);
    fn deserialize(src: &[?; Self::SIZE]) -> Self;
}
```

Oh and we don't know what the word of the serialized form is yet so that should be provided too:

```rust
trait SizedSerialize {
    const SIZE: usize;
    type Word;

    fn serialize(&self, dst: &mut [Self::Word; Self::SIZE]);
    fn deserialize(src: &[Self::Word; Self::SIZE]) -> Self;
}
```

Awesome!

```
error: generic parameters may not be used in const operations
  |
5 |     fn serialize(&self, dst: &mut [Self::Word; Self::SIZE]);
  |                                                ^^^^^^^^^^ cannot perform const operation using `Self`
  |
  = note: type parameters may not be used in const expressions

error: generic parameters may not be used in const operations
  |
6 |     fn deserialize(src: &[Self::Word; Self::SIZE]) -> Self;
  |                                       ^^^^^^^^^^ cannot perform const operation using `Self`
  |
  = note: type parameters may not be used in const expressions
```

...oh, not awesome.

Actually this error leads us into designing this interface to be more versataile anyway. We shouldn't lock implementors into necessarily using arrays as the serialization medium, they should be able to use whatever type they want:

```rust
trait SizedSerialize {
    type Serialized;

    fn serialize(&self, dst: &mut Self::Serialized);
    fn deserialize(src: &Self::Serialized) -> Self;
}
```

Nice, but we should constrain `Serialized` a bit so we can use it in the way we want, i.e. having a size. To accomplish this, let's make another trait:

```rust
trait Medium {
    type Word;
    const SIZE: usize;
}
```

And update `SizedSerialize`:

```rust
trait SizedSerialize {
    type Serialized: Medium;

    fn serialize(&self, dst: &mut Self::Serialized);
    fn deserialize(src: &Self::Serialized) -> Self;
}
```

Ok and what about `UnsizedSerialize`? Well we no longer need to keep track of a size, only the word:

```rust
trait UnsizedSerialize {
    type Word;

    fn serialize(&self, dst: ?);
    fn deserialize(src: ?) -> Self;
}
```

And in terms of the function parameters, we could go for a `&[Self::Word]`, but that would unnecessarily add a requirement for contiguous memory, so let's use iterators instead:

```rust
trait UnsizedSerialize {
    type Word;

    fn serialize(&self, dst: impl IntoIterator<Item = Self::Word>);
    fn deserialize(src: impl IntoIterator<Item = Self::Word>) -> Self;
}
```

...with proper access and lifetimes:

```rust
trait UnsizedSerialize {
    type Word;

    fn serialize<'a>(&self, dst: impl IntoIterator<Item = &'a mut Self::Word>)
    where
        Self::Word: 'a;
    fn deserialize<'a>(src: impl IntoIterator<Item = &'a Self::Word>) -> Self
    where
        Self::Word: 'a;
}
```

> `IntoIterator` is used so that iterators can be moved or passed by `&mut`.

## Fallibility

One missing piece of our interface is that right now it looks infallible, but of course this shouldn't be the case. Imagine we are deserializing an enum with only two variants, and the byte we read corresponds to neither of them, we can't construct the enum as the data is **invalid**.

Across both of our interfaces, there are only two ways to fail:

1. Invalid data
1. Insufficient data

The latter of which only applies to `UnsizedSerialize` for obvious reasons.

Let's make our error types:

```rust
mod error {
    #[derive(Debug, Clone, Copy)]
    pub struct EndOfInput;

    #[derive(Debug, Clone, Copy)]
    pub struct Invalid;

    #[derive(Debug, Clone, Copy)]
    pub enum Error {
        EndOfInput,
        Invalid,
    }

    impl From<EndOfInput> for Error {
        fn from(_: EndOfInput) -> Self {
            Self::EndOfInput
        }
    }

    impl From<Invalid> for Error {
        fn from(_: Invalid) -> Self {
            Self::Invalid
        }
    }
}
```

Rather than making a single enum `Error` we create two additional unit structs to be used in the functions where only one error is possible (maximizing invariance).

First we'll update `SizedSerialize`:

```rust
trait SizedSerialize {
    type Serialized: Medium;

    fn serialize(&self, dst: &mut Self::Serialized); // infallible!
    fn deserialize(src: &Self::Serialized) -> Result<Self, error::Invalid>;
}
```

- `SizedSerialize::serialize` is infallible because the destination buffer is by definition the correct size.
- `SizedSerialize::deserialize` will also always be reading from a buffer of the correct size, but not necessarily the correct *data*, so an invalid error can occur.

```rust
trait UnsizedSerialize {
    type Word;

    fn serialize<'a>(&self, dst: impl IntoIterator<Item = &'a mut Self::Word>) -> Result<(), error::EndOfInput>
    where
        Self::Word: 'a;
    fn deserialize<'a>(src: impl IntoIterator<Item = &'a Self::Word>) -> Result<Self, error::Error>
    where
        Self::Word: 'a;
}
```

- `UnsizedSerialize::serialize` can be passed an iterator that is too small to hold the serialized data.
- `UnsizedSerialize::deserialize` can also be passed an iterator that is too small to hold the serialized data, *and* can hold invalid data.

## Finishing Touches

We can reduce the load of implementing our interface by coupling the two traits a bit more.

If a type implements `SizedSerialize`, *surely* it could implement `UnsizedSerialize` as well? Perhaps rather than thinking of these traits in terms of what kinds of *types* implement them, we can think of them in terms of what kind of serialization medium they interact with.

Let's rename `UnsizedSerialize` to `SerializeIter`, and `SizedSerialize` to `SerializeBuf`.

Let's also use our realization that `SerializeBuf` implies `SerializeIter` and add that constraint:

```rust
trait SerializeBuf: SerializeIter {
    type Serialized: Medium;

    fn serialize(&self, dst: &mut Self::Serialized);
    fn deserialize(src: &Self::Serialized) -> Result<Self, error::Invalid>;
}
```

One problem is that it is possible a single type may want to support being serialized to and from mediums of differeing word sizes. As it is now, our associated type `Serialized` locks types into only having one serialized form, so let's create a new trait that can be used as a generic constrain on all of our previous traits:

```rust
trait Encoding {
    type Word;
}
```

...and update our other traits:

```rust
trait Medium<E: Encoding> {
    const SIZE: usize;
}
```

```rust
trait SerializeIter<E: Encoding> {
    fn serialize_iter<'a>(&self, dst: impl IntoIterator<Item = &'a mut E::Word>) -> Result<(), error::EndOfInput>
    where
        E::Word: 'a;
    fn deserialize_iter<'a>(src: impl IntoIterator<Item = &'a E::Word>) -> Result<Self, error::Error>
    where
        E::Word: 'a;
}
```

```rust
trait SerializeBuf<E: Encoding>: SerializeIter<E> {
    type Serialized: Medium<E>;

    fn serialize_iter(&self, dst: &mut Self::Serialized);
    fn deserialize_iter(src: &Self::Serialized) -> Result<Self, error::Invalid>;
}
```

And now, we can actually provide default implementations for the functions of `SerializeBuf` since we can just use the functions from `SerializeIter` knowing that the buffer size is correct:

In order to do this, we first need to ammend `Medium` a bit to gain access to iterators of the provided buffers:

```rust
trait Medium<E: Encoding = Vanilla> {
    const SIZE: usize;

    fn get_iter<'a>(&'a self) -> impl Iterator<Item = &'a E::Word>
    where
        E::Word: 'a;
    fn get_iter_mut<'a>(&'a mut self) -> impl Iterator<Item = &'a mut E::Word>
    where
        E::Word: 'a;
}
```

Then add the default implementations:

```rust
unsafe trait SerializeBuf<E: Encoding = Vanilla>: SerializeIter<E> {
    type Serialized: Medium<E>;

    fn serialize_iter(&self, dest: &mut Self::Serialized) {
        // SAFETY: dependent on safety of trait implementation.
        // `Serialized` must be of sufficient length.
        unsafe { SerializeIter::serialize_iter(self, &mut dest.get_iter_mut()).unwrap_unchecked() };
    }

    fn deserialize_iter(src: &Self::Serialized) -> Result<Self, error::Invalid> {
        SerializeIter::deserialize_iter(&mut src.get_iter()).or_else(|err| match err {
            error::Error::Invalid => Err(error::Invalid),
            // SAFETY: dependent on safety of trait implementation.
            // `Serialized` must be of sufficient length.
            error::Error::EndOfInput => unsafe { unreachable_unchecked() },
        })
    }
}
```

It is now **unsafe** to implement `SerializeBuf` because the default implementations will result in **undefined behavior** if the implementor's computation of the size of `SerializeBuf::Serialized` is incorrect.

All done!

# Procedural Generation

Of course, users *should not* need to be implementing either of these traits themselves, the implementations should be **derived** from the type definitions themselves... a derive macro!

Let's create an encoding called `Vanilla` which will represent a simple sequential encoding scheme for bytes.

```rust
pub struct Vanilla;
impl Encoding for Vanilla {
    type Word = u8;
}
```

That was easy, now we need to create a derive macro to implement the serialization traits for types. It should look something like this:

```rust
#[derive(vanilla::SerializeIter, vanilla::SerializeBuf)]
struct Foo {
    a: u8,
    b: bool,
    c: i16
}
```

And then one can simply write:

```rust
let mut buf = <Foo as SerializeBuf>::Serialized::default();
let foo = {/* Foo from somewhere */};
foo.serialize_buf(&mut buf);
```

...and conversely:

```rust
let foo = Foo::deserialize_buf(&buf).unwrap();
```

So clearly the `vanilla::SerializeBuf` macro needs to figure out the size of the type given its contents.

And the `vanilla::SerializeIter` macro needs to actually generate the code for serializing and deserializing the type. In the case of a struct, this should just be calling `SerializeIter`'s functions on each field in order.

In the case of `Foo`:

```rust
// pseudo code
impl SerializeIter for Foo {
    fn serialize_iter(&self, dst: impl IntoIter) -> Result<(), Error> {
        let dst = dst.into_iter();

        self.a.serialize_iter(&mut dst)?;
        self.b.serialize_iter(&mut dst)?;
        self.c.serialize_iter(&mut dst)?;

        Ok(())
    }

    fn deserialize_iter(src: impl IntoIter) -> Result<Self, Error> {
        let src = src.into_iter();

        Ok(
            Self {
                a: u8::deserialize_iter(&mut src)?,
                b: bool::deserialize_iter(&mut src)?,
                c: i16::deserialize_iter(&mut src)?,
            }
        )
    }
}
```

Neat!

Here is the procedurally generated implementations for our `Foo`:

```rust
impl cookie_cutter::SerializeIter for Foo {
    fn serialize_iter<'a>(
        &self,
        dst:impl IntoIterator<Item =  &'a mut <cookie_cutter::encoding::vanilla::Vanilla as cookie_cutter::encoding::Encoding> ::Word>,
    ) -> Result<(), cookie_cutter::error::EndOfInput>
    where
        <cookie_cutter::encoding::vanilla::Vanilla as cookie_cutter::encoding::Encoding>::Word: 'a,
    {
        let mut dst = dst.into_iter();
        cookie_cutter::SerializeIter::serialize_iter(&self.a, &mut dst)?;
        cookie_cutter::SerializeIter::serialize_iter(&self.b, &mut dst)?;
        cookie_cutter::SerializeIter::serialize_iter(&self.c, &mut dst)?;
        Ok(())
    }
    fn deserialize_iter<'a>(
        src:impl IntoIterator<Item =  &'a<cookie_cutter::encoding::vanilla::Vanilla as cookie_cutter::encoding::Encoding> ::Word>,
    ) -> Result<Self, cookie_cutter::error::Error>
    where
        <cookie_cutter::encoding::vanilla::Vanilla as cookie_cutter::encoding::Encoding>::Word: 'a,
    {
        let mut src = src.into_iter();
        Ok(Self {
            a: <u8 as cookie_cutter::SerializeIter>::deserialize_iter(&mut src)?,
            b: <bool as cookie_cutter::SerializeIter>::deserialize_iter(&mut src)?,
            c: <i16 as cookie_cutter::SerializeIter>::deserialize_iter(&mut src)?,
        })
    }
}

unsafe impl cookie_cutter::SerializeBuf for Foo {
    type Serialized = [u8;
    <<u8 as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE+ <<bool as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE+ <<i16 as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE];
}
```

Enums are where things get a bit more interesting:

```rust
#[derive(vanilla::SerializeIter, vanilla::SerializeBuf)]
#[repr(u8)]
enum Bar {
    A,
    B(bool) = 0x10,
    C { a: u8, b: f32 } = 0x20,
}
```

...woah, what a crazy enum. This is certainly more variable in terms of code generation, let's start with the basics.

Every variant of an enum has a **tag**, which is a primitive (integer) that **repr**esents the variant in memory. The type of this tag is selected with the `#[repr(...)]` directive. The discriminant (`Variant = {some_number}`) allows for the tag to be specified.

So in serialized form, a classic enum with purely unit variants can easily be represented as an integer as selected by the repr directive. The codegen for this would be a match statement corresponding the tag to the variant for deserialization and the variant to the tag for serialization:

Deserialize:

```rust
match tag {
    0 => Bar::A,
    0x10 => Bar::B(/* not implemented yet */),
    0x20 => Bar::C {/* not implemented yet */},
}
```

Serialize:

```rust
match bar {
    Bar::A => 0,
    Bar::B(_) => 0x10,
    Bar::C { _ } => 0x20,
}
```

Then, for any associated types of the variants, they can be serialized/deserialized in order:

```rust
match tag {
    0 => Bar::A,
    0x10 => Bar::B(bool::deserialize_iter(...)?),
    0x20 => Bar::C { a: u8::deserialize_iter(...)?, b: f32::deserialize_iter(...)?},
}
```

> There are some other minor considerations omitted here, like non-literal discriminants, tuple unpacking, etc.

So what is the size of this enum? Each variant definitely takes up a different amound of space:

```
Bar::A: #|XXXXX
        ^
       tag
Bar::B: #|#XXXX
        ^ ^
      tag bool
Bar::C: #|#####
        ^ ^^
      tag u8 f32

#: 1 byte
X: unused
```

Well we need to treat the size of the enum as its *maximum* size. That way all variants fit.

It would be a waste of space though to skip over the leftover bytes when the value held is a smaller variant, so we will just expect the bytes of the next item to be immediately after. This means the size of any collection type (tuples, structs, arrays) ranges from the sum of the minimum size of each contained type to the sum of the maximum size of each contained type.

Here is what the procedurally generated implementations look like for our `Bar`:

```rust
impl cookie_cutter::SerializeIter for Bar {
    fn serialize_iter<'a>(
        &self,
        dst:impl IntoIterator<Item =  &'a mut <cookie_cutter::encoding::vanilla::Vanilla as cookie_cutter::encoding::Encoding> ::Word>,
    ) -> Result<(), cookie_cutter::error::EndOfInput>
    where
        <cookie_cutter::encoding::vanilla::Vanilla as cookie_cutter::encoding::Encoding>::Word: 'a,
    {
        let mut dst = dst.into_iter();
        const A_TAG: u8 = 0 + 0;
        const B_TAG: u8 = 0x10;
        const C_TAG: u8 = 0x20;
        match self {
            Self::A => cookie_cutter::SerializeIter::serialize_iter(&A_TAG, &mut dst),
            Self::B(v0) => {
                cookie_cutter::SerializeIter::serialize_iter(&B_TAG, &mut dst)?;
                cookie_cutter::SerializeIter::serialize_iter(v0, &mut dst)?;
                Ok(())
            }
            Self::C { a, b } => {
                cookie_cutter::SerializeIter::serialize_iter(&C_TAG, &mut dst)?;
                cookie_cutter::SerializeIter::serialize_iter(a, &mut dst)?;
                cookie_cutter::SerializeIter::serialize_iter(b, &mut dst)?;
                Ok(())
            }
        }
    }
    fn deserialize_iter<'a>(
        src:impl IntoIterator<Item =  &'a<cookie_cutter::encoding::vanilla::Vanilla as cookie_cutter::encoding::Encoding> ::Word>,
    ) -> Result<Self, cookie_cutter::error::Error>
    where
        <cookie_cutter::encoding::vanilla::Vanilla as cookie_cutter::encoding::Encoding>::Word: 'a,
    {
        let mut src = src.into_iter();
        const A_TAG: u8 = 0 + 0;
        const B_TAG: u8 = 0x10;
        const C_TAG: u8 = 0x20;
        let tag = <u8 as cookie_cutter::SerializeIter>::deserialize_iter(&mut src)?;
        match tag {
            A_TAG => Ok(Self::A),
            B_TAG => Ok(Self::B(
                <bool as cookie_cutter::SerializeIter>::deserialize_iter(&mut src)?,
            )),
            C_TAG => Ok(Self::C {
                a: <u8 as cookie_cutter::SerializeIter>::deserialize_iter(&mut src)?,
                b: <f32 as cookie_cutter::SerializeIter>::deserialize_iter(&mut src)?,
            }),
            _ => Err(cookie_cutter::error::Error::Invalid),
        }
    }
}

unsafe impl cookie_cutter::SerializeBuf for Bar {
    type Serialized = [u8; {
        let mut max = 0;
        if<<bool as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE>max {
            max =  <<bool as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE;
        }
        if<<u8 as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE+ <<f32 as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE>max {
            max =  <<u8 as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE+ <<f32 as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE;
        }
        max+ <<u8 as cookie_cutter::SerializeBuf> ::Serialized as cookie_cutter::medium::Medium> ::SIZE
    }];
}
```

# Conclusion

Now, when working on systems with multiple coupled devices, the binaries can have a common library containing all the shared command types to be exchanged. Even external devices, like a phone over BLE, could have these types and the serialization implementations embedded into the application via FFI.

The actual facilitation of message passing, however, is still a problem we have yet to solve!
