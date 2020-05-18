---
title: Converting bits to integers in Rust using generics
description: Real use case for Generics in Rust with some hidden tricks
layout: post
date: 2020-04-04 22:26:00 +0200
tags: [rust, generics]
comments: true
---

Let's imagine there's input from some device, that produces only zeros and ones and it needed to be converted into actual integers, something like

{% highlight rust %}
let bits: Vec<u8> = vec![0, 0, 0, 0, 0, 1, 0, 1];
let result: u8 = convert(&bits);
assert_eq!(result, 5);
{% endhighlight %}


In this article, I will show how Rust functions can be generalized using generics and which problem I faced.

## Naive solution

Here's my first and naive implementation:

{% gist e959810d86dd4019ae03e58f89237937 %}

No magic, doing bit shift, and then XORing result with each bit. If it is 1 it will be added to result, if it is zero, zero remains in the rightmost position.

But there's room for improvement. [`fold`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.fold) function is more idiomatic:

<script src="https://gist.github.com/citizen-stig/f73b0ebe08e70f1a138727ddcbbd8983.js"></script>


## Generic Approach

The last solution is quite good, but it only produces an output of type `u8`. What if we want to have `u32`, or `i64`. Rust has support of [Generic Data Types](https://doc.rust-lang.org/book/ch10-01-syntax.html) for cases like that, let's try!

We will need `PartialEq`, `BitXor`, `Shl` and `From<bool>` [traits](https://doc.rust-lang.org/book/ch10-02-traits.html) to define behavior inside convert function:

<script src="https://gist.github.com/citizen-stig/53111b72b14636f43dffd95a40666c65.js"></script>

But this implementation gives this error during compilation:

{% highlight rust %}
10 |         .fold(zero, |result, bit| (result << one) ^ bit)
   |                                   --------------- ^ --- &T
   |                                   |
   |                                   <T as std::ops::Shl>::Output
   |
   = note: an implementation of `std::ops::BitXor` might be 
                            missing for `<T as std::ops::Shl>::Output`
{% endhighlight %}

Hm, this doesn't seem right. According to trait declaration, it has `Output` defined as [Associated type](https://doc.rust-lang.org/rust-by-example/generics/assoc_items/types.html):

{% highlight rust %}
pub trait Shl<Rhs = Self> {
    type Output;

    fn shl(self, rhs: Rhs) -> Self::Output;
}
{% endhighlight %}

Based on documentation `BitXor` has an implementation for `bool`, which doesn't have implementation for output of `<<` operator. 

Associated type is syntactic sugar and thankfully, Rust allows putting constraints on generic function implementation:

<script src="https://gist.github.com/citizen-stig/6b0a93b477c09ab876367c7ff5c25bfe.js"></script>

It also has a couple of other fixes, but generally, this is a working solution for all integer types.

## Improvements

What if I want to add the following to my function:

* Check if the input vector is larger than the target integer
* Check if the vector has only ones and zeros
* Reduce memory footprint, by changing input vector always be u8, and the return type will be derived from caller definition.

The first thing we need to do, is to change output type from plain `T` to `Result<T, ConversionError>` where `ConversionError` is an enum, that holds all error types we have:

{% highlight rust %}
#[derive(Debug)]
pub enum ConversionError {
    Overflow,
    NonBinaryInput,
}
{% endhighlight %}

To check the size of a generic type we will use [`std::mem::size_of`]() like that:

{% highlight rust %}
if bits.len() > (std::mem::size_of::<T>() * 8) {
    return Err(ConversionError::Overflow);
}
{% endhighlight %}

Changing the input vector to always be `u8` as the smallest integer is a questionable change because it forces always explicitly declare a return type, but I believe it is a fair deal for reducing memory size. 

This is how it should be called now:

{% highlight rust %}
let bits: Vec<u8> = vec![0, 0, 0, 0, 0, 1, 0, 1];
let result: Result<u32, ConversionError> = convert(&bits);
assert_eq!(result.unwrap(), 5);
{% endhighlight %}

Function signature will change mainly in `From<boolean`> to `From<u8>`:

{% highlight rust %}
pub fn convert<T: PartialEq + From<u8> + BitXor<Output=T> + Shl<Output=T> + Clone>(
    bits: &[u8],
) -> Result<T, ConversionError> {
{% endhighlight %}

Checking that input has only zeros and ones is optional, but can be done with simple filtering and counting:

{% highlight rust %}
if bits.iter()
    .filter(|&&bit| bit != 0 && bit != 1).count() > 0 {
    return Err(ConversionError::NonBinaryInput);
}
{% endhighlight %}

The final solution with all latest changes:
<script src="https://gist.github.com/citizen-stig/6e8efd2ba3b2cca3681098d35e5f91b6.js"></script>


