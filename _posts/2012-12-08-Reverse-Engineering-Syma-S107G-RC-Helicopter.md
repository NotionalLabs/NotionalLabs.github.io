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

![Carrier Signal](/assets/images/2012-12-08_Carrier_Wave.jpg "Carrier Signal")

#### Preamble:
**High:** 2ms (2000us) / **Low:** 2ms (2000us) / **Period:** 4ms (2000us)

![Preamble](/assets/images/2012-12-08_Preamble.jpg "Preamble") 

#### Zero:
**High:** 0.3ms (300us) / **Low:** 0.3 (300us) / **Period:** 0.6ms (600us)

![Zero](/assets/images/2012-12-08_Zero.jpg "Zero") 

#### One:
**High:** 0.3ms (300us) / **Low:** 0.7ms (700ms) / **Period:** 1ms (1000us)

![One](/assets/images/2012-12-08_One.jpg "One") 


#### Test:

|         |         |
| ------------: |:--------------|
| ![One](/assets/images/2012-12-08_One.jpg "One")      | **High:** 0.3ms (300us) / **Low:** 0.7ms (700ms) / **Period:** 1ms (1000us) |


#### Footer:
**High:** 0.3ms (300us)

![Footer](/assets/images/2012-12-08_footer.jpg "Footer") 

I almost missed this, but there is in fact a footer pulse at the end of the packet 300 microseconds long followed by a long period of low signal until the next packet header.

#### Channels

The Syma 107G controller appears to support 2 “channels” so that two pilots can fly their heli’s at the same time without interfering with each other. Examining the behaviour of the transmitter when switching channels indicated that the only differences between the two is a) the packet transmission interval and b) a special bit in the control packet is flipped (more on this later). The image below shows the differences in packet Tx intervals when the channel select slider is flipped (channel 1 of the Logic analyser):

![Channel Flip](/assets/images/2012-12-08_ChannelFlip.jpg "Channel Flip")

The transmit interval for Channel A is **120ms** (start of a packet header to the start of the next packet), or 8.33 packets per second. The transmit interval for Channel B is **180ms**, or 5.55 packets per second.

So surprisingly, the carrier modulation frequency remains the same on both channels – not the most robust design, huh? I suspect the reason the transmit interval is increased is to reduce the likelihood of a collision of control packets from two controllers – if a packet collision does occur, the next control frames from each controller will almost definitely be out of phase with each other, so the chopper shouldn’t just fall out of the air (though in my experience, they do get a bit clumsy when you have two going at once…).



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

