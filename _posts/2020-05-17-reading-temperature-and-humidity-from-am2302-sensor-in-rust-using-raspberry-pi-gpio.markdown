---
title: Reading temperature and humidity from AM2302 sensor in Rust using Raspberry Pi GPIO
layout: post
date: 2020-05-17 21:20:00 +0200
tags: [rust, generics]
comments: true
---

# Devices

I use [AM2302](https://www.adafruit.com/product/393) sensor, which is wired version of DHT22 sensor and Raspberry Pi 3


# Connection

Utility [`pinout`](https://pinout.xyz) allows to get actual pin number for particular Raspberry pi model,
here is part of the output for Raspberry Pi 3 model:

{% highlight bash %}
   3V3  (1) (2)  5V
 GPIO2  (3) (4)  5V
 GPIO3  (5) (6)  GND
 GPIO4  (7) (8)  GPIO14
   GND  (9) (10) GPIO15
{% endhighlight %}


* Red wire goes to `pin 2` 5V. Most other examples use 3.3V pin, but I decided to use 5 to avoid low voltage issues.
* Black wire goest to `pin 9`, ground. Any other ground can be used.
* Yellow wire goest to `pin 7`, which reffered as GPIO4, so `4` will be used in the code


![AM2302 wiring](/assets/img/AM2302_RPI3_bb.png)


# Code

There are several options for working with GPIO interface in Rust:

* [gpio-cdev](https://github.com/rust-embedded/gpio-cdev)
* [rust-sysfs-gpio](https://github.com/rust-embedded/rust-sysfs-gpio)

I decided to use gpio-cdev package, as sysfs interface is going [to be deprecated](https://www.kernel.org/doc/Documentation/ABI/obsolete/sysfs-gpio)


## Prepare

Adding dependecy to `Cargo.toml`:


{% highlight toml %}
[dependencies]
gpio-cdev = "0.2"
{% endhighlight %}

Gpio-cdev has pretty clear and simple documentation so we can have this function to get line.

{% highlight rust %}
use gpio_cdev::{Chip, Line};

fn get_line(gpio_number: u32) -> Line {
    let mut chip = Chip::new("/dev/gpiochip0").unwrap();
    chip.get_line(gpio_number).unwrap()
}
{% endhighlight %}

`gpio_number` in my case is 4.

## Requesting data

According to the [documentation](https://cdn-shop.adafruit.com/datasheets/Digital+humidity+and+temperature+sensor+AM2302.pdf) to initialize data transfer from device, it should be set status to zero for about at about 1 - 10 milliseonds.

{% highlight rust %}
use std::{thread, time};
use gpio_cdev::{Chip, LineRequestFlags, Line};

fn do_init(line: &Line) {
    let output = line.request(
        LineRequestFlags::OUTPUT,
        HIGH,
        "request-data").unwrap();
    output.set_value(0).unwrap();
    thread::sleep(time::Duration::from_millis(3));
}
{% endhighlight %}

## Reading Data, Part 1

There are 2 options reading the data,

1. Pulling data manually using  line request `get_value`
2. Subscribing to events from the line, using `.events` method of the line

Let's try first approach as it is straight forward.

In following example we will read state for changes during 10 seconds

{% highlight rust %}
let contact_time = time::Duration::from_secs(10);
let input = line.request(
    LineRequestFlags::INPUT,
    HIGH,
    "read-data").unwrap();

let mut last_state = input.get_value().unwrap();
let start = time::Instant::now();

while start.elapsed() < contact_time {
    let new_state = input.get_value().unwrap();
    if new_state != last_state {
        let timestamp = time::Instant::now();
        println!("Change of state {:?} => {:?}", last_state, new_state);
        last_state = new_state;
    }
}
{% endhighlight %}

But how to check if all these changes are valid data, or if initiliazing works properly?

I prefer to stop here, and start from other end of this: decoding data, and then get back to reading and check if data is correct. 


## Decoding data

Here is example from documentation on how data is repesented:

{% highlight text %}
DATA=16 bits RH data+16 bits Temperaturedata+8 bits check-sum
Example: MCU has received 40 bits data from AM2302 as
         0000 0010 1000 1100 0000 0001 0101 1111   1110 1110
             16 bits RH data      16 bits T data   check sum

         Here we convert 16 bits RH data from binary system to decimal system,
         0000 0010 1000 1100  →             652
               Binary system     Decimal system
         
         RH=652/10=65.2%RH

         Here we convert 16 bits T data from binary system to decimal system,
         0000 0001 0101 1111   →            351
               Binary system     Decimal system

         T=351/10=35.1°C

         When highest bit of temperature is 1, 
         it means the temperature is below 0 degree Celsius.
         
         Example: 1000 0000 0110 0101, T= minus 10.1°C
                       16 bits T data
         
         Sum       = 0000 0010+1000 1100+0000 0001+0101 1111 = 1110 1110 

         Check-sum = the last 8 bits of Sum                  = 11101110
{% endhighlight %}

So let's express it in Rust!

I define `Reading` struct that will hold actual temperature and humidity


{% highlight rust %}
#[derive(Debug, PartialEq)]
pub struct Reading {
    temperature: f32,
    humidity: f32,
}
{% endhighlight %}

Data we receive is not always correct, so errors will happen, this enum will represent it:

{% highlight rust %}
#[derive(Debug, PartialEq)]
pub enum CreationError {
    // Wrong number of input bites, should be 40
    WrongBitsCount,

    // Something wrong with conversion to bytes
    MalformedData,

    // Parity Bit Validation Failed
    ParityBitMismatch,

    // Value is outside of specification
    OutOfSpecValue,
}
{% endhighlight %}

And to build reading we will need to define a constructor, that will take vector of bytes and converts it to Reading.

It checks:

* That there's only 40 bits
* That vector contains only 1s and 0s
* That checksum is correct
* That actual values are valid by specification

It also does all necessary conversion.

Note, that `convert` function that converts vector of 1s and 0s to integer is described in [my another article](https://citizen-stig.github.io/2020/04/04/converting-bits-to-integers-in-rust-using-generics.html)

Also, specificaion does not mention it explicitly, but check sum is counted with overflow.

{% highlight rust %}
impl Reading {
    pub fn from_binary_vector(data: &[u8]) -> Result<Self, CreationError> {
        if data.len() != 40 {
            return Err(CreationError::WrongBitsCount);
        }

        let bytes: Result<Vec<u8>, ConversionError> = data.chunks(8)
            .map(|chunk| -> Result<u8, ConversionError> { convert(chunk) })
            .collect();

        let bytes = match bytes {
            Ok(this_bytes) => this_bytes,
            Err(_e) => return Err(CreationError::MalformedData),
        };

        let check_sum: u8 = bytes[..4].iter()
            .fold(0 as u8, |result, &value| result.overflowing_add(value).0);
        if check_sum != bytes[4] {
            return Err(CreationError::ParityBitMismatch);
        }


        let raw_humidity: u16 = (bytes[0] as u16) * 256 + bytes[1] as u16;
        let raw_temperature: i16 = if bytes[2] >= 128 {
            bytes[3] as i16 * -1
        } else {
            (bytes[2] as i16) * 256 + bytes[3] as i16
        };

        let humidity: f32 = raw_humidity as f32 / 10.0;
        let temperature: f32 = raw_temperature as f32 / 10.0;

        if temperature > 81.0 || temperature < -41.0 {
            return Err(CreationError::OutOfSpecValue);
        }
        if humidity < 0.0 || humidity > 99.9 {
            return Err(CreationError::OutOfSpecValue);
        }

        Ok(Reading { temperature, humidity })
    }
}
{% endhighlight %}

And it is always a good idea to write tests for it. Here I will only show couple of them, the rest is available in the repository
{% highlight rust %}
#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn wrong_parity_bit() {
        let result = Reading::from_binary_vector(
            &vec![
                0, 0, 0, 0, 0, 0, 1, 0,  // humidity high
                1, 0, 0, 1, 0, 0, 1, 0,  // humidity low
                0, 0, 0, 0, 0, 0, 0, 1,  // temperature high
                0, 0, 0, 0, 1, 1, 0, 1,  // temperature low
                1, 0, 1, 1, 0, 0, 1, 0,  // parity
            ]
        );
        assert_eq!(result, Err(CreationError::ParityBitMismatch));
    }

    #[test]
    fn correct_reading() {
        let result = Reading::from_binary_vector(
            &vec![
                0, 0, 0, 0, 0, 0, 1, 0,  // humidity high
                1, 0, 0, 1, 0, 0, 1, 0,  // humidity low
                0, 0, 0, 0, 0, 0, 0, 1,  // temperature high
                0, 0, 0, 0, 1, 1, 0, 1,  // temperature low
                1, 0, 1, 0, 0, 0, 1, 0,  // parity
            ]
        );

        let expected_reading = Reading {
            temperature: 26.9,
            humidity: 65.8,
        };

        assert_eq!(result, Ok(expected_reading));
    }

}
{% endhighlight %}

Having all this allows to get back to reading actual data and check if it is correct.


# Reading Data, Part 2

How do I convert all these state changes to vector of bits?

According to documentation, actual values are represented by amount of time signal was in `1` state, where

* `0` means 26-28 milliseconds
* `1` means 70 milliseconds


But another [document by aosong.com](http://akizukidenshi.com/download/ds/aosong/AM2302.pdf) I've found shows this table:

| Parameter                | Min | Typical | Max | Unit | 
| ------------------------ |:---:| -------:|----:|-----:|
| Signal "0" high time     |  22 |      26 |  30 |   μS |
| Signal "1" high time     |  68 |      70 |  75 |   μS |

So considering error in measurements, I will take `35ms` as cutoff between 1 and zero.
Also, I want to separate reading data and parsing it, so no cpu cycles will be spent during receiving.

raw data" will be represented with `Event` structure, which will have timestamp and type: 

{% highlight rust %}
#[derive(Debug, PartialEq)]
enum EvenType {
    RisingEdge,
    FallingEdge,
}

#[derive(Debug)]
struct Event {
    timestamp: time::Instant,
    event_type: EvenType,
}

impl Event {
    pub fn new(timestamp: time::Instant, event_type: EvenType) -> Self {
        Event { timestamp, event_type }
    }
}
{% endhighlight %}

And to convert vector of events to vector 1 or 0s pretty handy `.windows` iterator method will be used

{% highlight rust %}
fn events_to_data(events: &[Event]) -> Vec<u8> {
    events
        .windows(2)
        .map(|pair| {
            let prev = pair.get(0).unwrap();
            let next = pair.get(1).unwrap();
            match next.event_type {
                EvenType::FallingEdge => Some(next.timestamp - prev.timestamp),
                EvenType::RisingEdge => None,
            }
        })
        .filter(|&d| d.is_some())
        .map(|elapsed| {
            if elapsed.unwrap().as_micros() > 35 { 1 } else { 0 }
        }).collect()
}
{% endhighlight %}

Read events function should be just record timestamp of the changes and change type:
{% highlight rust %}
fn read_events(line: &Line, events: &mut Vec<Event>, contact_time: time::Duration) {
    let input = line.request(
        LineRequestFlags::INPUT,
        HIGH,
        "read-data").unwrap();

    let mut last_state = input.get_value().unwrap();
    let start = time::Instant::now();

    while start.elapsed() < contact_time {
        let new_state = input.get_value().unwrap();
        if new_state != last_state {
            let timestamp = time::Instant::now();
            let event_type = if last_state == LOW && new_state == HIGH {
                EvenType::RisingEdge
            } else {
                EvenType::FallingEdge
            };
            events.push(Event::new(timestamp, event_type));
            if events.len() >= 83 {
                break;
            }
            last_state = new_state;
        }
    }
}
{% endhighlight %}

Events vector is created outside of this function, because request to device is done before, spending time creating new vector can lead to loss of events.


# Combining it all together and testing

in `main.rs` I will declare 2 function, `try_read`, which will get bits we were able to pull from the device and check they can be converted to the data.:

{% highlight rust %}
fn try_read(gpio_number: u32) -> Option<Reading> {
    let mut final_result = None;
    let all_data = push_pull(gpio_number);
    if all_data.len() < 40 {
        println!("Saad, read not enough data");
        return final_result;
    }
    for data in all_data.windows(40) {
        let result = Reading::from_binary_vector(&data);
        match result {
            Ok(reading) => {
                final_result = Some(reading);
                break;
            }
            Err(e) => {
                println!("Error: {:?}", e)
            }
        }
    }
    final_result
} 
{% endhighlight %}

And main function simply tries to read every 5 seconds:
{% highlight rust %}
fn main() {
    let gpio_number = 4;  // GPIO4  (7)
    let sleep_time = time::Duration::from_secs(5);
    for _ in 1..30 {
        println!("Sleeping for another {:?}, to be sure that device is ready", sleep_time);
        thread::sleep(sleep_time);
        match try_read(gpio_number) {
            Some(reading) => println!("Reading: {:?}", reading),
            None => println!("Unable to get the data"),
        }
    }
}
{% endhighlight %}

Output will look like this:

{% highlight bash %}
   Compiling rust_gpio v0.1.0 (/home/pi/workspace/rust_gpio)
    Finished dev [unoptimized + debuginfo] target(s) in 4.02s
     Running `target/debug/rust_gpio`
Sleeping for another 5s, to be sure that device is ready
Error: ParityBitMismatch
Error: ParityBitMismatch
Unable to get the data
Sleeping for another 5s, to be sure that device is ready
Error: ParityBitMismatch
Reading: Reading { temperature: 23.2, humidity: 37.7 }
Sleeping for another 5s, to be sure that device is ready
Error: OutOfSpecValue
Reading: Reading { temperature: 23.2, humidity: 68.2 }
Sleeping for another 5s, to be sure that device is ready
Error: ParityBitMismatch
{% endhighlight %}

As you might see, it is not always to possible to read the data, and often first attemp is ParityBitMisMatch

# Note on different approach

Pulling doesn't look like the most performant way of reading data, and subscribing to events `.events()` suppose to be more efficient, but with this approach I was unable to get enough data in callback. So this require further investigation.

# Conclusion

Making binary data more readable simplifies experimentation a lot!

Couple improvement can be done, such as:

* Skipping 1st bit if there's 41 bits was received, as it is signal bit
* Allocating space for vector of events upfront
* Making pin number to be command line argument
* More extensive testing

The full example is aviailable in GitHub Repository [citizen-stig/gpio-am2302-rs](https://github.com/citizen-stig/gpio-am2302-rs)



