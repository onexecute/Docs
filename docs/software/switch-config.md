---
template: main.html
---

<img src="https://raw.githubusercontent.com/ExpressLRS/ExpressLRS-Hardware/master/img/software.png">

!!! warning "WARNING" 
    **Put your arm switch on AUX1**, and set it as **~1000 is disarmed, ~2000 is armed**.

ExpressLRS has two options for how switches are transmitted: Hybrid and WideHybrid. **The switch mode can only be changed when a receiver is not connnected.** Switch mode is changed using Lua configuration, the user_define setting is no longer needed.

For both switch modes, the first switch (AUX1) is sent with every packet. Put. Your. Arm. On. AUX1. For the remaining 7 switches, one switch is sent with each packet (in addition to AUX1) and which switch is sent is rotated on each packet. 

1. The default choice is `HYBRID_SWITCHES_8` or just "Hybrid" where the switches are broken into different types

| Channel | Resolution | Frequency |
|---|---|---|
| AUX1 | 1-bit / 2-position | Every packet, ARM channel |
| AUX2-AUX7 | 3-bit / 2-position, 3-position, or 6-position | Round-robin |
| AUX8 | 4-bit / 16-position | Round-robin |

It takes 7 packets to send the complete set of switches before cycling back to AUX2.

2. The other choice is "WideHybrid" mode. It shares much in common with regular Hybrid mode, but with just two different types

| Channel | Resolution | Frequency |
|---|---|---|
| AUX1 | 1-bit / 2-position | Every packet, ARM channel |
| AUX2-AUX8 | 6 or 7-bit / 64 or 128-position | Round-robin |

It takes 8 packets to send the complete set of switches before cycling back to AUX2 (one more than Hybrid). WideHybrid uses the 8th slot to transmit extra data to the receiver, including the current transmitter power. This is the only switch mode which can show TPwr on the flight controller's OSD.

## AUX 1 (All Modes)

!!! warning "WARNING" 
    **Put your arm switch on AUX1**, and set it as **~1000 is disarmed, ~2000 is armed**.
    
AUX1 is the low-latency switch, sent with every packet, and only supports on/off (2-position) operation. ExpressLRS uses AUX1 to determine if your model is armed, and should set up on the transmitter as armed at high values.

## AUX 2-7 (Hybrid)

The majority of the aux channels in Hybrid mode, AUX2 - AUX7, work with 2-position switches, 3-position switches, or 6-position switches / selector buttons.

Approx. Channel Input (us) | Channel Output (us) | Ardupilot Mode
-- | -- | --
988 | 1000 | Mode1 (up position for 2-pos / 3-pos)
1192 | 1275 | Mode2
1398 | 1425 | Mode3
1500 | 1500 | Mode4 (center position in 3-pos)
1602 | 1575 | Mode4
1807 | 1725 | Mode5
2011 | 2000 | Mode6 (down position for 2-pos / 3-pos)

## AUX 8 (Hybrid)

AUX8 is the wide range channel, supporting 16 positions. You can stack all your modes in here, Bardwell style, or get some low-res camera pan action-- 180 degrees / 16 actually isn't terrible. Clever math-heads might note that there's no "center position" (1500us) in a 16-position switch, so using AUX8 with a 3-position switch means it will come out as 1533 at the flight controller.

Switch Position | Channel Output (us) | Switch Position | Channel Output (us)
--|--|--|--
0 | 1000 | 8 | 1533
1 | 1066 | 9 | 1600
2 | 1133 | 10 | 1666
3 | 1200 | 11 | 1733
4 | 1266 | 12 | 1800
5 | 1333 | 13 | 1866
6 | 1400 | 14 | 1933
7 | 1467 | 15 | 2000

## AUX 2-8 (Wide)

In Wide switch mode, AUX2-AUX8 are 7-bit (128 position) for telemetry ratios 1:128 through 1:8, or 6-bit (64 position) for 1:4 and 1:2. These behave more like traditional channels although with lower precision. You can tell you're operating in Wide mode when a switch in the middle position shows up as 1503 instead of 1500.

Wide switch mode also sends current transmitter power to the flight controller to display in the OSD. TPwr is only available at the flight controller in this mode.

Ardupilot Modes don't line up very well with the standard -100% (988us) to +100% (2012us) output range in OpenTX when using a 6-position selector as input. Both the first two and the last two positions get binned into Mode 1 and Mode 6 respectively. To get the full 6 Ardupilot modes, go to the Outputs page on the OpenTX model setup and set the min / max for the channels to -75% / +75%.

## FAQ

### Why do you keep saying "*put arm on AUX1*"?

For safety reasons.

AUX1 is sent with every packet going out, this is the most reliable way to be able to tell your model to disarm. If your arm switch is in another aux channel, it may be several packets before that switch is transmitted, and there's no guarantee that the RX will actually receive that packet. There's a non-trivial chance your model **may not ever disarm** if the link quality is low and it just so happens that the packet containing the arm switch is getting missed every time. Forcing the arm switch into every packet on AUX1 means that if **any** packet is received by ExpresLRS, it will disarm your model, not just a less than 1-in-7 chance.

It also protects against unintentional disarms caused by a corrupt packet changing the value of the arm switch to disarmed. Betaflight requires that 4x "disarm" commands are received before disarming to guard against this possibility. With arm on AUX1, a single corrupt packet can not disarm your model. With arm on AUX2-8, the one corrupt switch value will be sent 6 times before the value is refreshed, but the flight controller would have already disarmed by that point.

Also keep in mind that for ExpressLRS, ~1000us is the **disarmed** state and ~2000us is the **armed** state.

### I use a 3-pos switch for arm, this software is unusable

You can still use a 3-position switch to arm! AUX1 just needs to be 2-position, not the physical switch. Simply adjust the mixer in your transmitter so AUX1 works like an on-off switch. If your 3-position arm switch had a second function as well, such as enabling Blackbox, just use one of the other 7 AUX channels to send the switch on a second channel.

### Why can't I have switches / potentiometers with full 10-bit resolution?

A lot of the magic of ExpressLRS comes from its small packet sizes, there simply is not enough space to put all that data. There are tricks that can be done with interleaving sticks and switches packets but we believe sticks should be in every packet for the lowest control latency. Different switch modes with higher resolution may be added further down the road for setups where higher latency doesn't matter.

### What about "Normal" one bit switch mode?

In 1.0, there was also a switch mode where there were 8x 1-position switches sent in every packet. This mode was removed in 2.0 due to its unpopularity and the flash space was used for other features.

### Every time I change switch mode in Lua, it changes back!

The switch mode can only be changed when a receiver is not connnected, to ensure a consistency between the RX and TX's interpretetation of the switch data. Power down your receiver, wait for the "Telemetry Lost" callout, and the switch mode change will stick.
