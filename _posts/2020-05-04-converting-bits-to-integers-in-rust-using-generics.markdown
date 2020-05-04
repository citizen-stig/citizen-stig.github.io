---
title: Converting bits to integers in Rust using generics
layout: post
date:   2020-04-04 22:26:00 +0300
tags: [rust, generics]
---

Let's imagine there's input from some device, that produces only zeros and ones and it needed to be converted into actual integers, something like

{% highlight rust %}
let bits: Vec<u8> = vec![0, 0, 0, 0, 0, 1, 0, 1];
let result: u8 = convert(&bits);
assert_eq!(result, 5);
{% endhighlight %}


In this article I will show how Rust functions can be generalized using generics and which problem I personally faced.

## Naive solution

Here's my first and naive implementation:

<script src="https://gist.github.com/citizen-stig/e959810d86dd4019ae03e58f89237937.js"></script>

No magic, doing bit shift and then XORing result with each bit. If it is 1 it will be added to result, if it is zero, zero remain in right most position.

But there's a room for improvement. [`fold`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.fold) function is more idiomatic:

<script src="https://gist.github.com/citizen-stig/f73b0ebe08e70f1a138727ddcbbd8983.js"></script>


## Generic Approach

Last solution is quite good, but it only produces output of type `u8`. What if we want to have `u32`, or `i64`. Rust has support of [Generic Data Types](https://doc.rust-lang.org/book/ch10-01-syntax.html) for cases like that, let's try!

We will need `PartialEq`, `BitXor`, `Shl` and `From<bool>` [traits](https://doc.rust-lang.org/book/ch10-02-traits.html) to define behaviour inside convert function:

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

Based on documentation `BitXor` has implementation for `bool`, which doesn't have implementation for output of `<<` operator. 

Associated type is a syntaxic sugar and thankfully, Rust allows to put constraints on generic function implementaiton:

<script src="https://gist.github.com/citizen-stig/6b0a93b477c09ab876367c7ff5c25bfe.js"></script>

It also has couple other fixes, but generally this is working solution for all integer types.

## Improvements

What if I want to add follwing to my function:

* Check if input vector is larger, than target integer
* Check if vector has only ones and zeros
* Reduce memory footprint, by changing input vector always be u8, and return type will be derived from caller definition.

First thing we need to do, is to change output type from plain `T` to `Result<T, ConversionError>` where `ConversionError` is enum, that holds all error types we have:

{% highlight rust %}
#[derive(Debug)]
pub enum ConversionError {
    Overflow,
    NonBinaryInput,
}
{% endhighlight %}

To check size of generic type we will use [`std::mem::size_of`]() like that:

{% highlight rust %}
if bits.len() > (std::mem::size_of::<T>() * 8) {
    return Err(ConversionError::Overflow);
}
{% endhighlight %}

Changing input vector to always be `u8` as smalles integer is questinable change, because it forces always explicitly declare return type, but I believe it is fair deal for reducing memory size. 

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

Checking that input has only zeros and ones is really optional, but can be done with simple filtering and counting:

{% highlight rust %}
if bits.iter()
    .filter(|&&bit| bit != 0 && bit != 1).count() > 0 {
    return Err(ConversionError::NonBinaryInput);
}
{% endhighlight %}

All together:
<script src="https://gist.github.com/citizen-stig/6e8efd2ba3b2cca3681098d35e5f91b6.js"></script>
