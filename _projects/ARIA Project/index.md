---
layout: post
title: ARIA Project
description: A Raspberry Pi-powered room automation system that physically actuates wall switches via servo motors, controlled through a Telegram bot.
skills:
  - Python
  - Raspberry Pi
  - Fusion 360
  - FDM 3D Printing
  - FEA
  - Linux/SSH
  - Breadboarding
main-image: /ARIA.jpg
---

## Overview

ARIA (Automated Room Interface & Actuator) is a room automation system built around a Raspberry Pi Zero 2W that physically actuates three rocker-style wall switches — controlling lights, a fan, and an outlet — using servo motors. The system is operated remotely through a Telegram bot, with no smart switches or rewiring required.

---

## Problem Statement

Standard smart home systems require replacing existing switches or rewiring — neither of which is practical in a rental or dorm setting. ARIA solves this by sitting on top of existing switches, actuating them mechanically, leaving the underlying electrical system untouched.

---

## System Overview

{% include image-gallery.html images="aria-wiring.jpg" height="400" %}

The system consists of three main components:

- **Raspberry Pi Zero 2W** — runs the Telegram bot and controls GPIO pins
- **SG90 Metal Gear Servos** — physically press each switch
- **Custom 3D Printed Servo Brackets** — mount servos in position against each switch

---

## Mechanical Design

A custom cover plate was modeled in Fusion 360 as a full replacement for the existing 3-gang Decora wall plate. After modeling, printing was skipped — the existing plate provided sufficient mounting structure, so fabricating a replacement wasn't necessary.

The servo brackets were designed and printed to mount directly to the wall plate and position each servo against its switch.

To keep the installation clean, a wall-mounted enclosure was designed and printed to house the Raspberry Pi, breadboard, and wiring. A printed cover snaps over the enclosure, concealing the electronics while keeping everything accessible.

{% include image-gallery.html images="aria-bracket.jpg, aria-installed.jpg" height="400" %}

### Servo Arrangement

A key design challenge was that three switches needed six actuation directions but only three servos were available. This was solved with a shared-servo approach:

- **TOP_1_2 servo** (GPIO 17) — actuates the ON direction for switches 1 and 2
- **BOTTOM_1_2 servo** (GPIO 27) — actuates the OFF direction for switches 1 and 2
- **SIDE servo** (GPIO 22) — controls switch 3 (fan) in both directions independently

### Bracket Iteration

The brackets went through three print iterations. Early versions failed due to weak layer adhesion under actuation load. After the first bracket failure, FEA was run in Fusion 360 to identify weak points in the geometry. The analysis revealed stress concentrations at a sharp interior corner, which was addressed by adding a fillet to distribute the load. Print orientation was also changed so layer lines aligned with the direction of actuation force, improving inter-layer strength.

---

## Software

The bot runs `switch_bot.py` on the Pi using the `python-telegram-bot` library. Commands sent through Telegram trigger GPIO-controlled servo movements.

### Servo Control

Two functions handle servo actuation. The `on` parameter determines which duty cycle direction to use:
```python
def flip(servo, on):
    servo.ChangeDutyCycle(5 if on else 2)
    time.sleep(0.5)
    servo.ChangeDutyCycle(3.5)
    time.sleep(0.5)
    servo.ChangeDutyCycle(0)

def cflip(servo, on):
    servo.ChangeDutyCycle(2 if on else 5)
    time.sleep(0.5)
    servo.ChangeDutyCycle(3.5)
    time.sleep(0.5)
    servo.ChangeDutyCycle(0)
```

Signal is cut to 0 after returning to neutral to eliminate servo hum.

### Command Handlers
```python
async def lights_on(update, context):
    cflip(top_servo, True)
    await update.message.reply_text("Lights are on!")

async def lights_off(update, context):
    cflip(bottom_servo, False)
    await update.message.reply_text("Lights are off!")

async def outlet_on(update, context):
    cflip(top_servo, False)
    await update.message.reply_text("Outlet is on!")

async def outlet_off(update, context):
    cflip(bottom_servo, True)
    await update.message.reply_text("Outlet is off!")
```

### Bot Commands

| Command | Action |
|-------------|--------------|
| /lon | Lights on |
| /loff | Lights off |
| /fon | Fan on |
| /foff | Fan off |
| /pon | Outlet on |
| /poff | Outlet off |

### Deployment

The bot runs as a systemd service (`aria.service`), auto-starting on boot and restarting on crash.

---

## Demo

{% include youtube-video.html id="XXXXXXXXXXX" autoplay="false" %}

---

## Lessons Learned

- Print orientation matters as much as geometry — three failed brackets before reorienting fixed the strength issue
- systemd is the right way to run persistent Pi services; nohup breaks on SSH disconnect
- A shared-servo approach can reduce hardware cost without sacrificing control, if the switch geometry allows it

---
