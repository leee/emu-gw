# emu-gw
E&M Unit Gateway with COM port control for PCs

## Introduction

This generic hardware gateway was originally designed for use with pairing Zello
resources with external infrastructure resources together. In this case:

|ethereal                                |corporeal                                                                     |
|:---------------------------------------|:-----------------------------------------------------------------------------|
|Zello<br>(including Zello@Work or ZeFFR)|PC software running in Radio Gateway mode<br>but no hardware interface exposed|
|Motorola ASTRO25 Core                   |Motorola gateway<br>like the GGM8000 CCGW (Conventional Channel Gateway)      |

There is some prior work compatible with Zello like the CriticalRF SiteCAST
Gateway, but provides only a single channel interface (likely a design intent
meant for single physical donor radios connectivity, originally meant for
CriticalRF's own PTToIP solution) for (in this original author's opinion) too
high a cost.

Our device offers interface connectivity for up to 4 physical resources, whether
they be donor radios or other gateways. Use of E&M Type II signaling allows for
two signaling units like a CCGW or RGU to be connected back to back.

## Physical Connectivity and Pinouts

The device provides:

2x USB Type B receptacles used for:
* USB1: USB hub for 4x audio interfaces each with a mic input and line out
* USB2: USB hub for 2x USB to serial UART devices each with 2 channels

2 USB ports were exposed to be generous with power availability. While 1 USB 2.0
port could've been used, it would've needed to utilize the Battery Charging
specification, or needed to be USB 3.0 to provide up to 900 mA current or USB
3.1 to take advantage of power management and USB Power Delivery. For the sake
of simplicity, and because the LMR world probably isn't ready for USB Type-C
connectors, 2 USB 2.0 B ports were used instead.

VMware bare metal hosts support up to USB 3.0 devices so long as the host has
the controller hardware and modules to support them, so this would not be a
design restriction.

and 4x 8p8c modular connector receptacles, commonly used for RJ45, but used here
for wiring to each resource 8 conductor wiring for 4-wire audio and E&M Type II
signaling which requires 4 conductors. The pins for these receptacles are
numbered 1 to 8:
* left to right when looking at the receptacle tab down
* right to left when looking at the receptacle tab up

Pinout follows Cisco E&M documentation.

| pin|I/O<br>to/from gateway|signal|description                                                                |
|---:|---------------------:|-----:|:--------------------------------------------------------------------------|
|   1|out                   |SB<br>|Signal Battery<br>supposed to be -48 VDC but instead positive here         |
|   2|in                    |M <br>|Mic/Magneto/transMit<br>control IN TO gateway for COR by detecting SB      |
|   3|in                    |R <br>|Ring<br> 600 ohm impedance balanced audio input                            |
|   4|out                   |R1<br>|Ring-1<br>600 ohm impedance balanced audio output with T1                  |
|   5|out                   |T1<br>|Tip-1<br>600 ohm impedance balanced audio output with R1                   |
|   6|in                    |T <br>|Tip<br>600 ohm impedance balanced audio input with R                       |
|   7|out                   |E <br>|Ear/Earth/recEive<br>control OUT FROM gateway for PTT by pulling down to SG|
|   8|in                    |SG<br>|Signal Ground                                                              |

Tip and Rings can be interchanged as well as referred to as positive or
negative. This shouldn't matter since by spec these are balanced audio in/outs.
However, if the gateway in the future provides support for unbalanced audio,
then rings should be considered ground and a decision be made on if 3/4 is ring
and 5/6 is tip or if it should be the other way around, as documented in other
devices.

## Zello PC Application - Radio Gateway Mode (also Zello@Work Server client)

The software supports radio control in one of three ways:
* VOX (Voice-Operated Exchange) on RX and/or TX paths. No VAD (Voice Activity
  Detection), and generally never the greatest idea if one can help it.
* COM(RS232), on RX and/or TX paths. Further explained below.
* and CM108-based devices, for which support is "coming soon" in perpetuity.

The software supports controlling a radio interface over a COM port. The RX and
TX paths can use different COM ports, although it's better to just settle on one
for each resource, and while this hasn't been tested, it's plausible to control
RX and TX for two radios over one COM port.

|direction                             |               RS-232 signal (DE9 pin)|signal state                                            |
|:-------------------------------------|-------------------------------------:|:-------------------------------------------------------|
|TX<br>in Zello parlance: "PC to Radio"|RTS (7)<br>DTR (4)                    |high or low<br>high or low                              |
|RX<br>in Zello parlance: "Radio to PC"|CTS (8)<br>DSR (6)<br>RI (9)<br>CD (1)|high or low<br>high or low<br>high or low<br>high or low|

If one really desired, they could use RTS/CTS to control one radio and use
DTR/DSR to control the other. Or if they just had a receive-only or a mixed
resource setup, they could use 4 radios receive-only, with 2 of those radios
capable of transmit.

And for reference, RS-232 pinout:

|DE9 pin|signal                   |
|------:|:------------------------|
|      1|`DCD` Data Carrier Detect|
|      2|`RXD` Received Data      |
|      3|`TXD` Transmit Data      |
|      4|`DTR` Data Terminal Ready|
|      5|`GND` Ground             |
|      6|`DSR` Data Set Ready     |
|      7|`RTS` Request to Send    |
|      8|`CTS` Clear to Send      |
|      9|`RI ` Ring Indicator     |

We provide each resource their own COM port for PTT and COR/COS.

## Motorola CCGWs

GGM CCGWs supports conventional resources in one of three ways:
* analog resources controlled with E&M (Ear and Mouth, or Earth and Magneto)
* analog resources controlled with TRC (Tone Remote Control)
* digital resources connected with V.24 serial

For the sake of minimizing parts and board complexity, we will not be utilizing
TRC, as implementing such would require also including DSP hardware (or doing
TRC in-band signal processing in analog filtering and detection). Additionally,
we would not be able to take advantage of the extra benefits of TRC like 4-bit
channel steering and other radio control options anyways since Zello doesn't
expose that kind of functionality to user clients anyways, so the point is moot.

Motorola specifies the GGM only supports E&M interfaces using 4-wire analog
using E&M Type II control. DC-control or 2-wire audio is explicitly stated not
supported. We opted to use this signaling method in our device for this reason.

## Donor radio I/O disclaimers

Our device is compatible with donor radios if their I/O is wired correctly for
use with E&M. This comes with a lot of caveats.

For example, while Motorola APX consolettes support E&M out of the box, older
ASTRO consolettes only support TRC if one has the TRC board upgrade over the
standard AIB (Audio Interface Board).

Wiring directly to subscribers themselves also comes with their own caveats. For
example, take ASTRO25 XTL mobile subscribers which come with I/O on their "J2"
accessory connector. Some of the logic lines on this I/O connector are limited
to -05VDC operation, with some other inputs rated for 0-20VDC. An exception is
UARTA, used for serial instead of USB programming, which can tolerate up to
`\pm` 15VDC maximum. For audio control, we're interested in the following lines:
* AUX_MIC (J2 pin 23), an unbalanced audio input with 660 ohm DC and 560 ohm AC
  impedance, which expects 80 mV_rms for 60% audio deviation (but also supports
  300 mV_rms for "future APCO accessories", "APCO default", as opposed to
  "motorcycle use").
* RX_FILT_AUDIO (J2 pin 21), an unbalanced 300 mV audio line-level output. Also
  documented as "output voltage is approximately 100 mV_rms per 1 kHz of
  deviation." Impedance not specified. There is a 1.4 VDC bias. If one does not
  plan on wiring this to a balanced line driver and instead want to bypass the
  gateway's balanced line receiver and wire this directly to a sound card's
  unbalanced audio input, they will need to remove the DC bias. In future
  revisions of this device, we may provide a switch per channel to operate on
  balanced or unbalanced audio as well as the basic filtering needed to remove
  DC bias.
* PTT (J2 pin 16, floats high) asserted to ground (J2 pins 1 or 14) keys up the
  donor radio, and can be taken advantage of by treating PTT as E-lead and donor
  radio ground as SG (signal ground). This is fully compliant with the E&M Type
  II interface model.
* CHAN_ACT (J2 pin 13, floats low) pushes high to +5 VDC on "qualified" channel
  activity. However, using this as E-lead is not so simple.

MOTOTRBO mobile subscribers also have I/O with the same lines, and to take
advantage of the 5V that is pushed high on CHAN_ACT, one will need to power a
relay that closes the gateway-provided SB (signal battery) to the M-lead.

SB is typically supposed to be -48VDC, a facet of the telecommunications
industry where telephony uses positive-grounded power. However, in the interest
of simplifying our design and not having to add an inverting boost converter or
other circuitry to provide this voltage, we instead supply a positive voltage
on our SB lead that qualifies as a HIGH signal on our UARTs and expect that
the E&M controller on the other end of our wireline respects the standard by
asserting their signal by relay. Connecting two E&M units back to back and
having this expectation hold true is a benefit of using Type II or Type V E&M.

If one would like to simplify the components necessary on the donor radio end to
control the E-lead, if they can reference the same ground on both ends, the +5V
that the donor radio provides as CHAN_ACT can be used to assert that HIGH signal
condition on the E-lead and not even have to bother with our modified SB.

This is a common change which can also be done on conventional base radios like
Quantars that also asserts +5 VDC, and is implemented on devices like the
[Christine Wireless](https://christinewireless.com/) RIC-M (Radio
Internet-Protocol Communications Module).
[Their manual](https://www.acgsys.com/wp-content/uploads/2015/08/Advanced_RIC_M_8.13.15rev.pdf)
documents this on page 13.

In a future revision that allows the use of unbalanced audio, we could reference
everything to the same ground and also provision the jumpers/switches needed to
use a gateway provided positive SB or donor radio HIGH signal.

Regarding audio: even if we could implement support for 2-wire audio, 4-wire is
strongly preferred for full-duplex audio even though radios really operate in
half-duplex. This is for status tones like TPT (talk permit) or reject and TOT
(timeout timer). While Zello doesn't provide a full-duplex audio experience on
user clients, it could be implemented by them in the future.

## Temporary BOM
```
sub      unit   qt part
 8.6900  2.1725 4x CMEDIA CM108B
12.2900  6.1450 2x FTDI (FUTURE DESIGNS) FT2232D-REEL
 2.8726  1.4363 2x MICROCHIP TECH USB2514B-AEZC-TR
 2.1720  0.5430 4x TOSHIBA TLP172GM(TPL,E(O
24       6      4x TI DRV134/DRV135 (price overestimated)
16       4      4x TI INA134/INA137 (price overestimated)
 6.2475  6.2475 1x AMPHENOL ICC RJHSE538404
 2.2210  1.1105 2x AMPHENOL ICC UE27BC54100
19.0000 19.0000 1x HAMMOND 1455L801 (from Newark)
93.4931
```

Maybe look at cheaper 8p8c connectors.
```
 0.7366  0.7366 1x BOOMELE(Boom Precision Elec) RJ45-B-1*4
```

Look at connectors with magnetics (like MagJack) and think about if they would
work or not.

JLCPCB's promo supports up to 10x10cm PCBs. The 1455L801 enclosure selected
at the moment is meant for 10cm wide boards, but if you want PCB end plates,
they need to be 103mm wide. going beyond that size won't be too expensive
though.

Make front and rear panels with PCBs with cutouts for ports, and silkscreening
for light indication legend, port numbering, and maybe pinout guide too.
