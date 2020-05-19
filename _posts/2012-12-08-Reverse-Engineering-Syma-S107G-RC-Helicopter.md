---
layout: post
title: "Hardware Reversing: Syma S107G RC Helicopter"
description: An exploration of the IR control protocol used by the S107G toy helicopter, and my first foray into hardware reverse engineering.
comments: true
category: reverse\ engineering
tags: infrared protocol reversing saleae logic
---

*[2020-05-18] This post was migrated from my old blog in 2012. I've tried to ensure all links, images, and content are reproduced, but please @ me if you find anything missing.*

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

|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||
|----|:-|
|![Trim Dial](/assets/images/2012-12-08_trim_dial.jpg "Trim Dial"){:height="245px" width="152px"}|You may remember that I said above that everyone else who has reverse-engineered the protocol arrived at a 4-byte control packet.
So where was my 4th byte? Well there’s one more dial on the controller we haven’t discussed – Trim.
Trim is used to calibrate the rotor speed balance in order to account for any idiosyncrasies in the helicopter’s build that cause it to have a rotational bias to left or right.|

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

## Conclusion

So there you have it – my first foray into hardware hacking and exploration. I think I’ve arrived at a definitive physical protocol definition, with timing values even better, perhaps, than the other projects I’ve mentioned throughout.  I put this down to the use of a logic analyser applied directly to the controller – this was unaffected by delays caused by the rise/fall times of the LEDs, IR detector, etc… that were part of the receiver circuits used by Kerry Wong and Hamsterworks to get around the issue of demodulating the 38khz IR signal. I was able to use Saleae’s excellent software to overcome those issues (albeit a little manually – it won’t demodulate the signal for you to my knowledge). The Saleae Logic also allowed me to sample at 16Mhz, certainly fast enough to get a high-resolution picture of the signals and really get some tight timing values for the transitions.

I hope this helps some new hacker get to grips with some of the ideas at play in reverse-engineering and hardware hacking, it’s certainly been a journey of discovery for me. If anyone has any questions, feel free to get in touch.

## UPDATE – 12-8-2012

I’ve had a little time this weekend to compare a 4-byte controller with 3-byte controller, so here are my findings.

* The timings for the symbols and transmission intervals are identical between the two controllers, so my timings should work equally well on any revision of the 107G you end up getting.
* One way to distinguish between which protocol version the controller is using may be the silkscreened model number and date on the controller PCB. The 3-byte controller had the model “S107T1” and the date “20090818” printed underneath the yaw/pitch and throttle control, respectively. The 4-byte controller had “s107T2” and “20100308” printed under the yaw/pitch control and on the right of the PCB.
* The fourth byte indeed  communicates the value of the Trim, but I actually identified something really interesting – the offsetting of the Yaw value occurs anyway, regardless of packet type! Take a look at this:

```
Control:         Control packet (4 byte)              Yaw:    Trim:
Trim 100% Left:  01010000 00111111 10110100 01111011   80      123
Trim Centre:     00111101 00111111 11001001 00111000   61      56
Trim 100% Right: 00101101 00111111 11001111 00000001   45      1
```

I suspect that the 107G doesn’t actually do anything with the Trim packet (perhaps I’ll test that with my DIY controller sometime soon), which explains how it’s able to handle both packet types – it simply ignores anything after the 3rd byte. I suspect that the 4th byte may be used by other toys using a similar design.

## UPDATE – April 6th, 2014

I finally got round to finishing the post containing an actual implementation of the protocol using an Arduino Uno. You can find it [here](/misc/protocols/Syma107_ProtocolSpec_v1.txt).