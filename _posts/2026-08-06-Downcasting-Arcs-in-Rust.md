---
layout: post
title:  "Downcasting Arcs in Rust"
date:   2026-08-06
visible: 1
---

The [`Arc`](https://doc.rust-lang.org/std/sync/struct.Arc.html) type is Rust's thread-safe smart pointer, and like other pointer types in Rust (e.g., [`Box`](https://doc.rust-lang.org/std/boxed/struct.Box.html)), one can use the `as` keyword[^1] to cast an `Arc<MyStruct>` into an `Arc<dyn MyTrait>`, as long as `MyTrait` is [dyn compatible](https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility).

What about casting it back from `Arc<dyn MyTrait>` to `Arc<MyStruct>`? Well, here the plot thickens a bit. We can't use the `as` keyword here, because that operation is inherently unsafe. An arbitrary `Arc<dyn MyTrait>` might not hold a `MyStruct` in it, but some other object that implements `MyTrait`, and trying to cast it into the wrong type would result in undefined behaviour. But the fact that something is unsafe doesn't mean it should be impossible, or even undesirable.

{%
    include image.html src='assets/img/arc/unmask.jpg'
    alt="A meme from Scooby Do, where a masked figure has the caption \"Arc<dyn MyTrait>\" and after it's unmasked, its caption is \"Arc<MyStruct>\""
%}

<!--more-->

## Why would I want to do that?

`Arc` has a few nice functions such as [`into_inner`](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.into_inner) that allow you to consume it if you're the only owner and unwrap the contained value. This is not possible for unsized types (such as `dyn MyTrait`) since they can only reside on the heap. So if we have an `Arc<dyn MyTrait>` that we're 100% sure was originally created as an `Arc<MyStruct>`, and we want to extract that `MyStruct` out, we can't do that unless we first convert the `Arc` back to an `Arc<MyStruct>`.[^2]

So I'd like to implement the following function:
```rust
/// Calling this is only safe if `arc` was originally created
/// as an `Arc<MyStruct>`.
unsafe fn downcast(arc: Arc<dyn MyTrait>) -> Arc<MyStruct>
```
[^3]

## Is there no simple solution?

Currently, `Arc` has support for downcasting from [`dyn Any`](https://doc.rust-lang.org/std/any/trait.Any.html) through the fallible [`downcast`](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.downcast). There's also an unconditional downcasting in nightly, in the form of [`downcast_unchecked`](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.downcast_unchecked).

But what if my trait isn't a subtrait of `Any`? We obviously can't implement the casting safely, but there should still be an unsafe way to achieve what we want. We just need to use the available API a bit more creatively.

## Raw pointer conversions

The `Arc` type provides the functions [`into_raw`](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.into_raw) and [`from_raw`](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.from_raw) to convert `Arc`s to and from raw pointers (`*const T`).

What would happen if we convert an `Arc<dyn MyTrait>` into a pointer, and then convert that pointer to an `Arc<MyStruct>`? Is this safe? Let's check the docs.

The docs for `Arc<T>::from_raw` have a lengthy list of safety requirements, the important ones for our case are:

* _The raw pointer must have been previously returned by a call to `Arc<U>::into_raw`_

Note that we have two types here, `U` and `T`, the two types aren't required to be identical, so we're good on this front.

* _If `U` is unsized, its data pointer must have the same size and alignment as `T`. This is trivially true if `Arc<U>` was constructed through `Arc<T>` and then converted to `Arc<U>` through an unsized coercion._

The last sentence describes our situation exactly, so we're good here as well!

All we need to do is implement our function:

```rust
/// Calling this is only safe if `arc` was originally created
/// as an `Arc<MyStruct>`.
unsafe fn downcast(arc: Arc<dyn MyTrait>) -> Arc<MyStruct> {
    let ptr = Arc::into_raw(arc);
    unsafe { Arc::from_raw(ptr.cast()) }
}
```

## Wait, why aren't we done?

Before we go on our merry way, let's check what the function actually compiles into, specifically in comparison to the upcast. We use [`Compiler Explorer`](https://rust.godbolt.org/z/so1nov1rE) with the following code:

```rust
use std::sync::Arc;

pub struct MyStruct;
pub trait MyTrait {}

impl MyTrait for MyStruct {}

#[unsafe(no_mangle)]
pub fn upcast(arc: Arc<MyStruct>) -> Arc<dyn MyTrait> {
    arc as _
}

#[unsafe(no_mangle)]
pub unsafe fn downcast(arc: Arc<dyn MyTrait>) -> Arc<MyStruct> {
    let ptr = Arc::into_raw(arc);
    unsafe { Arc::from_raw(ptr.cast()) }
}
```

to get this assembly output (with my own comments)

```nasm
upcast:
        ; Copy the input pointer into the first output register
        mov     rax, rdi
        ; Load a pointer to MyStruct's virtual table into the second
        ; output register
        lea     rdx, [rip + .Lanon.09a60105b64a7b096223e4c3559521b0.0]
        ret

; MyStruct's virtual table
.Lanon.09a60105b64a7b096223e4c3559521b0.0:
        .asciz  "\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\001\000\000\000\000\000\000"

downcast:
        ; Uhhhh...
        mov     rax, qword ptr [rsi + 16]
        ; wat?
        dec     rax
        ; why?
        and     rax, -16
        ; huh?
        add     rax, rdi
        ret
```

The upcast is very simple, it just converts a regular pointer into a fat pointer - a pointer + metadata, which in the case of a `dyn` object is another pointer, to the original type's virtual table.

But the downcast managed somehow to become significantly more complicated with all kinds of unexpected arithmetic instructions, instead of just copying the pointer part of the input into the output with a single `mov rax, rdi` instruction.

We can't leave it like that, right?

## What is going on?

In order to understand what's going on, we first need to look at the (highly edited) innards of `Arc<T>`:

```rust
struct Arc<T> {
    ptr: NonNull<ArcInner<T>>,
    // Stuff ...
}

#[repr(C)]
struct ArcInner<T: ?Sized> {
    strong: Atomic<usize>,
    weak: Atomic<usize>,
    data: T,
}
```

So `Arc<T>` is simply a pointer to an `ArcInner<T>`, which is two atomic `usize`s (8 bytes each) followed by our `T`[^4]. But here is the catch! There might be a gap between `weak` and `data`, depending on the [alignment](https://doc.rust-lang.org/reference/type-layout.html#r-layout.properties.align) of `T`.

Our `T`'s alignment requires its address to be a whole multiple of some power of 2, which, in turn, forces `ArcInner`'s address to also be a whole multiple of the same power.

The atomic counters in `ArcInner` already force it to have a minimum alignment of 8, so if `T` was itself a `usize` with an alignment of 8 as well, the end of `weak` would be divisible by 8 and would be a valid address for `data`.

```
+-------+-------+-------+
|strong | weak  |data   |
+-------+-------+-------+
0       8      16      24
```

If, however, `T` had an alignment of 32, we'd need a padding of 16 bytes between `weak` and `data`, to make sure its address divides by 32:

```
+-------+-------+---------------+
|strong | weak  |   <padding>   |
+-------+-------+---------------+
0       8      16              32
+-------------------------------+
|             data              |
+-------------------------------+
32                             64
```

Now, remember that `Arc<T>` is simply a pointer to `ArcInner<T>`, so it points to the beginning of the above diagrams. That means that the offset between the `Arc`'s pointer and the pointer to the `data` (which is what we get when we call `Arc::into_raw`) depends on `T`, or more specifically, on the alignment of `T`.

## Understanding the downcast, take 2

Armed with a new perspective, we can decipher the assembly code for our `downcast` function:

```nasm
downcast:
        ; Load the alignment of the input type from the
        ; input virtual table
        mov     rax, qword ptr [rsi + 16]

        ; Round down to the previous multiple of 16,
        ; giving the padding in `ArcInner`
        dec     rax
        and     rax, -16

        ; Add the result to the input pointer!?
        add     rax, rdi
        ret
```
[^5]

If the alignment is smaller or equal to 16, rounding down will net you a padding of 0, and any other alignment `a` will net you a padding of `a-16` as expected.

But why are we adding the padding to the `Arc`'s pointer - the address of `ArcInner`? That makes no sense at all!

## Understanding the downcast, take 3

The culprit this time is the compiler's optimisations, and we can fight them by splitting our function into two parts, the `into_raw` and the `from_raw`:
```rust
#[unsafe(no_mangle)]
pub fn downcast_into_raw_part(arc: Arc<dyn MyTrait>) -> *const MyStruct {
    Arc::into_raw(arc).cast()
}

#[unsafe(no_mangle)]
pub unsafe fn downcast_from_raw_part(ptr: *const MyStruct) -> Arc<MyStruct> {
    unsafe { Arc::from_raw(ptr) }
}
```

Suddenly new assembly lines appear:

```nasm
downcast_into_raw_part:
        ; Load the alignment of the input type from the
        ; input virtual table to the output register
        mov     rax, qword ptr [rsi + 16]

        ; Round down to the previous multiple of 16,
        ; giving the padding in `ArcInner`
        dec     rax
        and     rax, -16

        ; Add the input pointer to the result
        add     rax, rdi

        ; Add 16 to the result
        add     rax, 16
        ret

downcast_from_raw_part:
        ; Load the input pointer minus 16 to the output register
        lea     rax, [rdi - 16]
        ret
```

Both parts make sense now:
1. `into_raw` adds the padding + 16 (the total size of the `strong` and `weak` counters) to the base address of the `ArcInner`, to give us the address of `data`.
2. `from_raw` subtracts 16 from the address of the `data` to get us the address of the `ArcInner`. It doesn't need to subtract any padding, since it knows it's an `Arc<MyStruct>`, and `MyStruct` has an alignment of 1 and therefore needs no padding.

When the two parts come together, the last instruction of the `into_raw` part (add 16) and the first instruction of the `from_raw` part (subtract 16) cancel each other, and the compiler throws them out, which is how we ended up with the weird code above.

The fact that `from_raw` doesn't need to subtract any padding means that `into_raw` doesn't need to do that either. You know that, I know that, but it appears nobody told this to the compiler.

Nothing in the API informs the compiler that our `arc: Arc<dyn MyTrait>` input variable is guaranteed to contain a `MyStruct`, which has an alignment of 1.

## Speaking compilerish

Rust has a very powerful interface to convey information to the compiler in the form of the [`assert_unchecked`](https://doc.rust-lang.org/std/hint/fn.assert_unchecked.html) hint.

If we assert that the alignment of the `data` in the `Arc<dyn MyTrait>` is equal to the alignment of `MyStruct`, the compiler _should_ have enough information to remove the unnecessary arithmetic:

```rust
#[unsafe(no_mangle)]
pub unsafe fn downcast_with_hint(arc: Arc<dyn MyTrait>) -> Arc<MyStruct> {
    unsafe {
        std::hint::assert_unchecked(
            std::mem::align_of_val(arc.as_ref()) ==
            std::mem::align_of::<MyStruct>(),
        )
    };
    let ptr = Arc::into_raw(arc);
    unsafe { Arc::from_raw(ptr.cast()) }
}
```

Does it work? Yes, it does!

```nasm
downcast_with_hint:
        ; Copy the input pointer into the output register
        mov     rax, rdi
        ret
```

The arithmetic is gone, and our function is finally as simple as it should be.

## Will I use this?

Probably not. But that's not the point!

I mostly went down this rabbit hole because I was annoyed that an operation which should have been trivial wasn't. In the process, I learned a bit about the internals of `Arc` and alignments[^6], and maybe you did too :)

---

[^1]: An [unsized coercion](https://doc.rust-lang.org/reference/type-coercions.html#unsized-coercions) if you want the fancy term.

[^2]: That's not an entirely hypothetical situation. The [`arrow`](https://docs.rs/arrow/latest/arrow/) crate's API uses `Arc<dyn Array>` everywhere, whose reference can be safely downcast to specific types (e.g., `&Int32Array`) by using the `Array::as_any` function. But if we want an `Int32Array` on the stack (to use one of its consuming functions, such as [`into_parts`](https://docs.rs/arrow/latest/arrow/array/type.Int32Array.html#method.into_parts)), our only option is to clone that reference. Admittedly not a very expensive operation, but definitely avoidable if we hold the only reference. See [this issue](https://github.com/apache/arrow-rs/issues/8794) for instance.

[^3]: The entire post is relevant for [`Rc`](https://doc.rust-lang.org/std/rc/struct.Rc.html) as well, I just happened to investigate this stuff using `Arc` since I use it much more often.

[^4]: The `#[repr(C)]` keeps the order of the fields in memory identical to their order in the declaration.

[^5]: The alignment is read from `rsi + 16`, which is the 16th byte of the virtual table. If you look at the binary data of the virtual table in the assembly output, you'll find that the 16th byte has a value of `1`, which is exactly `MyStruct`'s alignment as a zero-sized struct.
    ```nasm
    ; MyStruct's virtual table
    .Lanon.09a60105b64a7b096223e4c3559521b0.0:
            .asciz  "\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\001\000\000\000\000\000\000"
    ```

[^6]: As well as LLVM IR and MIR, which I decided to spare from the reader.
