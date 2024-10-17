---
title: "The 2024 Capstone Chronicles"
date: 2024/10/17
categories:
  - projects
  - software
  - hardware
tags:
  - capstone
  - embedded development
share: true
toc: true
toc_sticky: true
---

## Concept:
My capstone project is simple: **I want to connect a real car's instrument cluster up to a video game (particularly *BeamNG.drive*) so I can experience the "thrill" of "racing".** The inspiration was *"I thought it would be cool*", so I absolutely sent it and registered for the capstone program. So here we are!

Okay, enough yapping. How am I going (and currently trying) to accomplish this?
## Project Preparations:
First, I had to see if this was even doable. So, after some poking around, I discovered that the popular car simulator game *BeamNG.drive* has a modding API with support for CAN Bus data operations. I checked out the logistics, and it seemed pretty straightforward, so I decided to commit to the project and ordered/obtained my supplies:
### The Instrument Cluster:
I needed a digital cluster with CAN support to make this happen, so after a short trip to the scrap yard and a couple of hours of guess-and-check later, I managed to scrounge up a 2011 VW Jetta instrument cluster. With no way of knowing whether or not it worked on-site, I took my trophy to the registers and p(r)aid. Thankfully, when I got home and tested it out with my voltage generator, I managed to get it working pretty quickly. So, with that, my primary pain point was over and done with, and I was free to move on to other things.
### The CAN Interface:
I didn't realize how expensive these things are. $340 for a USB adapter??? Anyways, my wallet is used to being empty, so I forked it up and waited a few days for my good to arrive. Surely, it came, so I unboxed it and started tinkering around. 

Now in theory, minus some bits and bobs, that should be all of the supplies... so now let's try and make it work!

## The Process:
Once I had the cluster powered, the first thing I did was attach some m/f jumper cables from the respective ports on my interface to my CAN high/low cables from the cluster.

After some pesky baud rate-related errors and general stupidity on my part, I managed to get a stable CAN connection with the cluster, and was free to observe all of the meaningless hexadecimal whizzing across my screen.

I started doing research on how to actually translate the information I was receiving, and quickly stumbled upon a CAN database for the platform of car I'm working with. The database is basically a map of what each CAN ID correlates with logistically, so it guided me in the right direction when trying to parse the raw data coming out of the bus.
### Interpreting CAN Data:
#### The Odometer:

I started low and slow with something easy--the odometer (mileage) readout. Given this is something I can easily verify by powering the cluster on, I decided to try calculating the value using the bitwise math I was given in the database:

```python
0x520: b7 << 8 << 8 + b6 << 8 + b5 -> mileage
```
*(0x520 is one of many CAN IDs that store information about the current state of the vehicle, and in this case, we can see that performing some bitwise arithmetic on the bits in this byte gives us the mileage calculation)*

In my case, the value of `0x520` read `0x57 0xC4 0x00 0x00 0x80 0x6D 0x20 0x03`, so I just had to do the calculation above to calculate the current mileage readout:

```python
0x03 << 8 << 8 + 0x20 << 8 + 0x6D
= 204909
```

Perfect! That's exactly right. Now I knew that this ID table was at least *somewhat* correct, so I moved forward with it and started working on other things, such as...
#### The Tachometer:
In my quest for infinite power, I needed to ensure that the cluster was receiving the right signals to operate, meaning I needed to mimic the behavior of the car using CAN signals. One of the signals I was looking at was `0x280`, which according to my ID table had bits pertaining to the torque and RPM of the engine. I knew that the car would likely need this information to display any sort of speedometer info, so I transmitted some test data over the channel to try and spoof some values.

Theoretically, all I had to do was follow what the ID table told me:

```python
0x280:  
	b0 -> clutch pedal
	b1 = b4 = b7 -> torque
	b3 << 8 + b2 -> RPM
	b5 -> cruise/accelerator pedal
```

So I crafted the following payload to send, with the goal of reading out 3000 RPM:

```python
0x01 0x50 0xB8 0x0B 0x50 0x00 0x00 0x50
```

I sent the payload, and anxiously waited... for the needle to immediately spin to ~700RPM and stay stagnant there. Huh.

After some more experimenting and tweaking of values, I discovered that the following payload ACTUALLY led to 3000RPM on the dot:

```python
0x01 0x50 0xFF 0x2D 0x50 0x00 0x00 0x50
```

But why? Well, here's what those b2 and b3 bits translate to:

```python
0xFF = 255
0x2D = 45
```

I realized something here--that b3 bit was an angle. A perfectly-correlated 45-degree angle relative to halfway between the bounds of the gauge, which was 3000RPM exactly. So if I set b3 to some angle between 0 and 90 degrees, the tachometer needle will jump to that position.

What a strange way to do things. But it'll have to work. So, to prove my hypothesis, I decided to make the needle sit halfway between 3000RPM and 6000RPM (the max number).

I converted 68 degrees into hexadecimal, which gave me `0x44`, and sent the payload off again with the new information. And to my pleasant surprise, I was correct in my assumption, and the needle quickly settled exactly where I had told it to go. I love it when things just work! (*unlike any Linux distro I've tried... sorry nerds*)