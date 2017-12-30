+++
title = "QR encoding"
author = "Daan van der Kallen"
draft = true
slug = "qr-encoding"
date = "2017-12-28"
description = ""
keywords = ["Development", "encoding", "QR"]
tags = ["Development", "encoding", "QR"]
+++


## Intro
For a project we had to put text in QR-codes, seemed simple enough, but sadly
the printers that our customers used only supported the
[alphanumeric-type QR-code](https://en.wikipedia.org/wiki/QR_code#Storage)
which has a small available character set of 45 characters
(`0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ $%*+-./:`). But we wanted to be able to
send unicode strings which require a much larger character set.

So it quickly became apparent that we needed some kind of encoding. The most
well-known encoding is probably [Base64](https://en.wikipedia.org/wiki/Base64)
however sadly this uses too many characters for our goal. However with a few
modifications we can still use it's concept.

# Base45
Base64 distributes 3 bytes over 4 characters with a character size of set 64.
This works because 3 bytes can have \\( 256^3 = 16,777,216 \\) different values 
and the 4 characters can have \\( 64^4 = 16,777,216 \\) values as well.

In our case we just need to use 45 instead of 64 and choose a nice number of
bytes. You can actually pretty easily calculate how much characters you need
for a certain amount of bytes. \\( \\log\_{45} 256^n \\) does the trick. We can
see that this works for Base64 as well since \\( \\log\_{64} 256^3 = 4 \\).

Let's try this for a few \\( n \\) for our case of 45 characters:

 \\( n \\) | \\( \\log\_{45} 256^n \\)     
-----------|---------------------------
         1 |         1.456703203759506 
         2 |         2.913406407519012 
         3 |         4.370109611278518 
         4 |         5.826812815038024 
         5 |         7.283516018797530
         6 |         8.740219222557036 
         7 |        10.196922426316542 
         8 |        11.653625630076048 
         9 |        13.110328833835554 
        10 |        14.567032037595060 

Sadly we don't get any nice round numbers like Base64 does. (This is actually
impossible since the prime factors of 256 and 45 can't match (\\(256 = 2^8\\), 
\\(45 = 3^2 \\times 5\\)). However we don't need the amount of values to match
exactly, as long as our characters can contain at least as many different
values as the bytes it will work.

However increasing the amount of characters used decreases the
efficiency of our encoding. An encodings efficiency is determined by the amount
of characters divided by the amount of bytes. We can thus create a new table
to investigate this further:

 \\( n \\) | characters | efficiency         
-----------|------------|--------------------
         1 |          2 | 2                  
         2 |          3 | 1.5                
         3 |          5 | 1.6666666666666667 
         4 |          6 | 1.5                
         5 |          8 | 1.6                
         6 |          9 | 1.5                
         7 |         11 | 1.5714285714285714 
         8 |         12 | 1.5                
         9 |         14 | 1.5555555555555556 
        10 |         15 | 1.5                

We can see that the best efficiency we get is 1.5, it can get lower with more
bytes with a theoretical limit of \\( \\log\_{45} 256 = 1.4567... \\) but this 
would require so many bytes that it becomes unpractical. 1.5 is in this case 
close enough. It is then easiest to choose the fewest amount of bytes and thus 
our best option seems to be to encode 2 bytes in 3 characters.

Another thing that we have to think about is what happens when our amount of
bytes is not dividable by 2. We chose to just represent a single remaining byte
with 2 characters instead of 3.

## Encoding example
Start with a string

`Foobar`

turn into list of bytes using UTF8

`[70, 111, 111, 98, 97, 114]`

turn into groups of 2 bytes

`[[70, 111], [111, 98], [97, 114]]`

turn into integers treating groups as base 256

`[18031, 28514, 24946]`

turn into groups of 3 integers smaller than 45

`[[8, 40, 31], [14, 3, 29], [12, 14, 16]]`

turn into list of integers smaller than 45

`[8, 40, 31, 14, 3, 29, 12, 14, 16]`

turn into encoded string

`8+VE3TCEG`

## Decoding example
Start with an encoded string

`96%DV E2K44VE40DVS0X`

turn into list of integers smaller than 45

`[9, 6, 38, 13, 31, 36, 14, 2, 20, 4, 4, 31, 14, 4, 0, 13, 31, 28, 0, 33]`

turn into groups of 3 integers smaller than 45

`[[9, 6, 38], [13, 31, 36], [14, 2, 20], [4, 4, 31], [14, 4, 0], [13, 31, 28],
[0, 33]]`

turn into integers treating groups as base 45

`[18533, 27756, 28460, 8311, 28530, 27748, 33]`

turn into groups of 2 bytes

`[[72, 101], [108, 108], [111, 44], [32, 119], [111, 114], [108, 100], [33]]`

turn into list of bytes

`[72, 101, 108, 108, 111, 44, 32, 119, 111, 114, 108, 100, 33]`

turn into string using UTF8

`Hello, world!`

