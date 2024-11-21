# [Bluetooth](https://www.bluetooth.com/) Protocol Implementation

## Goal
Agent implements the bluetooth protocol for communicating with IOT devices.

## Why Bluetooth?

Bluetooth is a proocol used across a wide variety of devices ranging all the way from from headphones to thermometers and scales.
 That is to say, supporting the bluetooth protocol opens up Akri for interfacing with a very wide range of devices used both commerically and privately.
 As bluetooth implements [piconets](https://en.wikipedia.org/wiki/Piconet) and [scatternets](https://en.wikipedia.org/wiki/Scatternet) implementing bluetooth for akri allows end users to create more complex use cases by using a single bluetooth device as a communication point from Akri to talk to a potentially larger network of bluetooth devices.
 In practice this could be a collection of something like thermometers or cameras that talk to a single monitoring device which in turn would talk to Akri.

## Background

_TODO_

## Disocvery Process

_TODO_

## Broker interfacing

_TODO_

## Security Considerations

_TODO_

## Outstanding Questions
- Do different devices implement the bluetooth protocol differently?
- Should we limit our scope to bluetooth low energy (BLE) devices/
- What does pairing look like?
    - How does authentication work?
    - Can we only discover devices that are advertising?
    - What is bluetooth's default mode?
    - If pairing must occur does Akri need to enter some retry logic?

## Misc.

### Library Audits

#### BTLEPlug and BlueR

|  |stars|forks|watches|issues|tests|
|--|-----|-----|-------|------|-----|
|**btleplug**|303|60|15|34|[/]|
|**bluer**|58|117|7|4|[x]|

| | [bluer](https://github.com/bluez/bluer)| [btleplug](https://github.com/deviceplug/btleplug/) |
| - | ---- | ---- |
|stars| 58 | 303 |
|forks| 11 | 60 |
|watches| 7 | 15 |
|issues| 4| 34 |
| contributors | 22 | 43 |
|tests| [N](https://github.com/bluez/bluer/issues/4) | Y |
| platfrom | Linux | w10,Osx,Linux,ios|
| bt classic support | kinda | N |

## Notable issues
- BlueR:
    - [No unit / integratin tests](https://github.com/bluez/bluer/issues/4)
    - [Only officially built for amd64](https://github.com/bluez/bluer/issues/3)
- BTLEPlug:
    - meant to be a host only
    - [what about bluer](https://github.com/deviceplug/btleplug/issues/257)
        - would be a replacement to dependencies on [bluez-async](https://github.com/bluez-rs/bluez-async) however the architecture difference is not something that the btleplug people feel like figuring out right now
