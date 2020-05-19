---
layout: post
title: "Hardware Reversing: Syma S107G RC Helicopter"
description: An exploration of the IR control protocol used by the S107G toy helicopter, and my first foray into hardware reverse engineering.
comments: true
category: reverse\ engineering
tags: infrared protocol reversing saleae logic
---

*[2020-05-18] This post was migrated from my old blog in 2012. I've tried to ensure all links, images, and content are reproduced, but please @ me if you find anyuthing missing.*

The Syma 107 RC helicopter is a pretty great gadget, all things considered – it’s cheap, fun, and ridiculously good for the price. It’s also a spectacular gadget for fledgling hackers.

![Syma S107G Helicopter](/assets/images/2012-12-08_S107G-title.jpg "Syma S107G Helicopter")

**1-26-2013 UPDATE – I’ve written up a full protocol specification for the Syma 107G – If you are here looking for a purely technical description of the protocol, [look here](/misc/protocols/Syma107_ProtocolSpec_v1.txt)** 

I first discovered a series of articles about this RC Heli on Hackaday, where a number of people had developed proof-of-concept controllers or jammers based on a decoded IR protocol for controlling the chopper. As it happened, these Heli’s were a bit of a fad in our lab and we had a number of them lying around (both alive and dead) so naturally I decided to have a go myself. I initially created a prototype Arduino IR controller that uses Processing to provide a front-end for one helicopter, then expanded it to control two similtaneously by switching the channel bits rapidly. I won’t go on too much about this here because what I actually want to talk about is a quirk in the protocol I decoded.

So there’s a decent body of work in the hacker and RC communities to decode the protocol used by the Syma 107 – here are a couple:

  * [This post](http://www.kerrywong.com/2012/08/27/reverse-engineering-the-syma-s107g-ir-protocol/) by Kerry Wong is a really great read and a study in methodical analysis in reverse engineering.

  * [This site](http://hamsterworks.co.nz/mediawiki/index.php/FPGAheli) by Hamsterworks is also a great practical guide to reverse engineering the protocol and I actually used these timings for my own controller a while back. *Link is no longer online*

Notably, one thing these projects both have in common is that they arrived at a protocol specification that uses 32-bits (4 bytes).

## Reverse Engineering the Controller

So enter my new Saleae Logic Analyser . I’ve never used a logic analyser or oscilloscope before, so I wanted to start with a project that I knew was possible and would have an output that I already understood (or so I thought…).

![Saleae Logic 8](/assets/images/2012-12-08_SaleaeLogic.jpg "Saleae Logic 8")

*As an aside, the Saleae Logic is really an excellent tool – I had read a lot about it and there are any number of superlatives floating around, but I must say it is EXACTLY what I was looking for. Great price-point, loads of features and the software is superb. If you are curious about logic analysers and don’t want to spend too much, I really recommend the Logic – it’s been great for me so far.*

I noticed immediately that the board in my controller was markedly different than the ones pictured on either the above links. Mine has way more components for a start:

![Controller with Logic Analyzer harness](/assets/images/2012-12-08_circuitboard_with_logic.jpg "Controller with Logic Analyzer harness")

I began by using a multimeter to ensure that the unlabelled IC wasn’t operating at a voltage higher than 5v and wouldn’t accidentally damage the Logic analyser. Everything was fine so I began probing the pins. Here is a rough guess at the pinout (note: I only really care about the pin that drives the IR LED transmitter circuit so I never looked into confirming how the control input works, etc…):

![Controller IC pinout](/assets/images/2012-12-08_IC_Pinout.jpg "Controller IC pinout")

Now that I know which pin the transmission signal is output on (pin 8), I can sniff it and see the 32-bit control protocol right? Well, almost. I should explain a little about the protocol for those who are unfamiliar.

## Decoding the physical packet format

The following timings are based on observations I’ve made on a sample of control packets captured with the logic analyser. Here’s a picture of the full packet:

![Example packet](/assets/images/2012-12-08_Example_Packet_3.jpg "Example packet") 

### Carrier Modulation

**Modulation Freq**: 38khz (50% duty cycle, 26us period, so 13us high/13us low)

38khz is a common carrier frequency for consumer Infrared communications. For the uninitiated, the white blocks in the main image above are made up of the high-frequency oscillations you can see on the left. This is usually done so that various transmitters that use the same medium (in this case the Infrared light spectrum – probably around the 940nm wavelength) can transmit on a different carrier modulation frequency and not interfere with the others’ transmissions (this is called [F]requency Division Multiplexing](http://en.wikipedia.org/wiki/Frequency_division_multiplexing)).

### Symbols
There are 4 types of symbol in the packet format: Preamble, Zero (“0”), One (“1”), and a footer:

![Carrier Signal](/assets/images/2012-12-08_Carrier_Wave.jpg "Carrier Signal"){: .center-image }

#### Preamble:

|-|:-|
|![Preamble](/assets/images/2012-12-08_Preamble.jpg "Preamble")| **High:** 2ms (2000us) / **Low:** 2ms (2000us) / **Period:** 4ms (2000us) |

#### Zero:

|-|:-|
|![Zero](/assets/images/2012-12-08_Zero.jpg "Zero")| **High:** 0.3ms (300us) / **Low:** 0.3 (300us) / **Period:** 0.6ms (600us) |

#### One:

|-|:-|
| ![One](/assets/images/2012-12-08_One.jpg "One") | **High:** 0.3ms (300us) / **Low:** 0.7ms (700ms) / **Period:** 1ms (1000us) |

#### Footer:

|-|:-|-|-|
| ![Footer](/assets/images/2012-12-08_footer.jpg "Footer") | **High:** 0.3ms (300us) | | |

I almost missed this, but there is in fact a footer pulse at the end of the packet 300 microseconds long followed by a long period of low signal until the next packet header.

### Channels

The Syma 107G controller appears to support 2 “channels” so that two pilots can fly their heli’s at the same time without interfering with each other. Examining the behaviour of the transmitter when switching channels indicated that the only differences between the two is a) the packet transmission interval and b) a special bit in the control packet is flipped (more on this later). The image below shows the differences in packet Tx intervals when the channel select slider is flipped (channel 1 of the Logic analyser):

![Channel Flip](/assets/images/2012-12-08_ChannelFlip.jpg "Channel Flip"){: .center-image }

The transmit interval for Channel A is **120ms** (start of a packet header to the start of the next packet), or 8.33 packets per second. The transmit interval for Channel B is **180ms**, or 5.55 packets per second.

So surprisingly, the carrier modulation frequency remains the same on both channels – not the most robust design, huh? I suspect the reason the transmit interval is increased is to reduce the likelihood of a collision of control packets from two controllers – if a packet collision does occur, the next control frames from each controller will almost definitely be out of phase with each other, so the chopper shouldn’t just fall out of the air (though in my experience, they do get a bit clumsy when you have two going at once…).

## The Control Packet Structure

So now that we understand what a packet looks like, let’s decode one and have a look at it at a higher level:

![Packet Decode](/assets/images/2012-12-08_packet_decode.jpg "Packet Decode"){: .center-image }

 And here it is again:

```
 |     byte 0    |     byte 1    |     byte 2    |
  0 0 1 1 1 1 1 1 0 0 1 1 1 1 1 1 0 1 0 1 0 0 1 0
 |  Decimal: 63  |  Decimal: 63  |  Decimal: 82  |
```
So we have a header, followed by 3 bytes (24 bits) of information, plus a footer pulse that makes up a control packet. Now lets figure out what the data means by trying various permutations of the controls (zero throttle, full throttle, 100% left turn, 100% right turn, etc…) and see how the data changes:


```
Type:         Data:                     Decimal Values (byte #):
100% Throttle 001111110011111101111111  (0): 63  (1): 63  (2): 127
50% Throttle  001111110011111100111111  (0): 63  (1): 63  (2): 63
100% Left     011111100011111101000110  (0): 126 (1): 63  (2): 70
100% Right    000001100011111101001011  (0): 6   (1): 63  (2): 75
100% Forward  001111110000000001010100  (0): 63  (1): 0   (2): 84
100% Back     001111110111011101010100  (0): 63  (1): 119 (2): 84
Channel A     001111110011111101010010  (0): 63  (1): 63  (2): 82
Channel B     001111110011111111010010  (0): 63  (1): 63  (2): 210
```

*Note: The controller requires that there be at least some throttle applied before it will send any packets, which is why byte 2 appears to change a little on the other tests.*

From this information, we can make the following assumptions:

**Byte 0**

* Byte 0 represents the Yaw (left/right) control.
* Byte 0 has a range of 0-127.
* 0-62 is a right turn, 63 is centre (default) value, 64-127 is a left turn.
* the first bit, bit 0, appears to always be 0.

**Byte 1**

* Byte 1 represents the Pitch (forward/backwards) control.
* Byte 1 has a range of 0-127.
* 0-62 is pitch forward, 63 is centre (default) value, 64-127 is pitch backwards.
* the first bit appears to always be 0.

**Byte 2**

* Byte 2 represents throttle and channel.
* Byte 2 (throttle) has a range of 0-127.
* 0 is 0% throttle, 127 is 100% throttle.
* the most significant bit of Byte 2 indicates which Channel is selected. 0 for Channel A, 1 for Channel B. This is how we can have 2 channels without changing the carrier frequency.

## The Mystery of the Missing Byte

|-|:-|
|![Trim Control](/assets/images/2012-12-08_trim_dial.jpg "Trim Control")|You may remember that I said above that everyone else who has reverse-engineered the protocol arrived at a 4-byte control packet. So where was my 4th byte? Well there’s one more dial on the controller we haven’t discussed – Trim. Trim is used to calibrate the rotor speed balance in order to account for any idiosyncrasies in the helicopter’s build that cause it to have a rotational bias to left or right.|

In the other’s work, their controller sent a fourth byte containing information about the Trim dial’s setting. My controller doesn’t have that byte, but I do have Trim control… so what’s the deal? Let’s do a bit more testing:

```
Type:         Data:                     Decimal Values (byte #):
Trim 100% L   010100000011111101010100  (0): 80  (1): 63  (2): 84
Trim Centre   001111110011111101010100  (0): 63  (1): 63  (2): 84
Trim 100% R   001011100011111101010100  (0): 46  (1): 63  (2): 84
```

We can see that the only byte that changes is Byte 0, so we can assume that rather than have a whole separate packet for trim control, the controller simply offsets the Yaw by a as much as -17 (right/clockwise compensation) or +17 (left/counter-clockwise compensation).

From a reverse engineering point of view, this is an interesting quirk. I have tried and tested both kinds of remote with the same helicopters and they both work flawlessly. What’s more, before I decoded the protocol myself, I built a whole controller based on the 4-byte protocol and never had any issues. So how does the helicopter know which protocol to use?

I think it might have something to do with the footer. The 3-byte protocol doesn’t require the helicopter to do anything in order to account for trim, while the 4-byte protocol requires the chopper’s on-board controller to offset the Yaw by the Trim value. The footer appears to consist a 300us high pulse then nothing until the next header, and this appears to be the case even on the other chaps’ 4-byte packets too. I suspect that if the helicopter’s on-board controller detects the end of a packet and it has only received 3-bytes of data, it pads the trim register with zeroes (no supplied trim offset) because it can assume trim control has been applied by the handset controller. Just a guess, but it seems logical. In any case, who would have thought the helicopter would be backwards/forwards compatible?

# Header 1

Here is some text

## Header 2

Here is some other text

### Header 3

Here is yet more text

#### Header 4

What the F.

# Some emphasis styles:

Emphasis, aka italics, with *asterisks* or _underscores_.

Strong emphasis, aka bold, with **asterisks** or __underscores__.

Combined emphasis with **asterisks and _underscores_**.

Strikethrough uses two tildes. ~~Scratch this.~~

# Lists

1. First ordered list item
2. Another item
  * Unordered sub-list. 
1. Actual numbers don't matter, just that it's a number
  1. Ordered sub-list
4. And another item.

⋅⋅⋅You can have properly indented paragraphs within list items. Notice the blank line above, and the leading spaces (at least one, but we'll use three here to also align the raw Markdown).

⋅⋅⋅To have a line break without a paragraph, you will need to use two trailing spaces.⋅⋅
⋅⋅⋅Note that this line is separate, but within the same paragraph.⋅⋅
⋅⋅⋅(This is contrary to the typical GFM line break behaviour, where trailing spaces are not required.)

* Unordered list can use asterisks
- Or minuses
+ Or pluses

# Links

[I'm an inline-style link](https://www.google.com)

[I'm an inline-style link with title](https://www.google.com "Google's Homepage")

[I'm a reference-style link][Arbitrary case-insensitive reference text]

[I'm a relative reference to a repository file](../blob/master/LICENSE)

[You can use numbers for reference-style link definitions][1]

Or leave it empty and use the [link text itself].

URLs and URLs in angle brackets will automatically get turned into links. 
http://www.example.com or <http://www.example.com> and sometimes 
example.com (but not on Github, for example).

Some text to show that the reference links can follow later.

[arbitrary case-insensitive reference text]: https://www.mozilla.org
[1]: http://slashdot.org
[link text itself]: http://www.reddit.com

# Images

Here's our logo (hover to see the title text):

Inline-style: 
![alt text](https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Logo Title Text 1")

Reference-style: 
![alt text][logo]

[logo]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Logo Title Text 2"

# Code

Inline `code` has `back-ticks around` it.

```javascript
var s = "JavaScript syntax highlighting";
alert(s);
```
 
```python
s = "Python syntax highlighting"
print s
```
 
```
No language indicated, so no syntax highlighting. 
But let's throw in a <b>tag</b>.
```

{% highlight ruby %}
def foo
  puts 'foo'
end
{% endhighlight %}

# Tables

Colons can be used to align columns.

| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |

There must be at least 3 dashes separating each header cell.
The outer pipes (|) are optional, and you don't need to make the 
raw Markdown line up prettily. You can also use inline Markdown.

Markdown | Less | Pretty
--- | --- | ---
*Still* | `renders` | **nicely**
1 | 2 | 3

# Blockquotes

> Blockquotes are very handy in email to emulate reply text.
> This line is part of the same quote.

Quote break.

> This is a very long line that will still be quoted properly when it wraps. Oh boy let's keep writing to make sure this is long enough to actually wrap for everyone. Oh, you can *put* **Markdown** into a blockquote. 

