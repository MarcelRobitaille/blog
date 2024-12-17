+++
title = "I thought I was so clever controlling my 3D printer from Python, until this…"
date = 2021-12-12
[extra]
header_image = "/endstop-calibration/poster.png"
[taxonomies]
tags= ["3d_printing", "programming", "python"]
+++

This is the story of a recent failure of mine.
Nobody was hurt an nothing was broken, but it was humiliating and a big waste of my time.

Anyone who has calibrated a [delta 3D printer](https://reprap.org/wiki/Delta_geometry) with the manual method
knows how tedious it can be.
After setting up a bed probe on my printer,
a process I discuss in my previous two blog posts,
I wanted to automate this calibration by commanding my printer through [Python](https://www.python.org/).
I wasted my time.

<!-- more -->

## Delta Background and Theory

Delta printers do not have independent X, Y, and Z axes,
but rather 3 vertical towers in an equilateral triangle that work together to move the print head in 3D space.
To determine the initial position, which is lost when the machine loses power,
on most delta machines, each tower is driven all the way to the top until it actuates a switch.
This processing is called homing.

Since the plane formed by all these switches is not perfectly parallel to the bed,
we typically do a calibration and enter the error of each tower into the printer's firmware.
It can then move the axis by this amount after homing, which represents its calibrated zero position.
These errors can be added to the constant `DELTA_ENDSTOP_ADJ` (delta endstop adjustment)
in [Marlin's](https://marlinfw.org/) `Configuration.h`.
The constant should be set to a list of three values
representing the [X, Y, and Z tower](https://reprap.org/wiki/Delta_geometry#Delta_columns_and_axis_names) error in millimeters in that order.
For example, for me the line should look like this:
```c++
#define DELTA_ENDSTOP_ADJ { -4.3, -2.0, -0.8 }
```
Alternatively, if your printer has an EEPROM,
you can set the endstop adjustment using G-Code [`M666 [X<adj>] [Y<adj>] [Z<adj>]`](https://marlinfw.org/docs/gcode/M666.html),
for example: `M666 X-4.3 Y-2.0 Z-0.8`.

This calibration is typically done manually by moving the nozzle down until it is close enough to the bed
that it is possible to move a piece of paper underneath with some resistance, but not too much.
This is done near each of the towers, and the height of the nozzle is recorded.
Then, the recorded values are used to update `DELTA_ENDSTOP_ADJ`,
the printer is homed, and the process repeats until the resistance is good with the nozzle at the same height for each tower.
This is tedious, subjective, and usually takes several iterations.

## Automating the calibration with a probe

This process is so excruciating for me that I would often spend a few minutes
looking online for better ways to do it,
procrastinating before this boring task.
For some reason, I never found any guides on an automated way.
Was I searching the wrong thing (looking back, this is probably the reason)?
Was it because most delta printers don't have bed probes?

When I finally got a bed probe, there was no way I was using the paper trick again.
If I couldn't find software to calibrate this with my probe,
I would just write my own!
I found the ["Single Z-Probe" Marlin G-Code command (`G30`)](https://marlinfw.org/docs/gcode/G030.html),
which probes the bed at a given XY position.
I would just need to write some software to do this near each tower,
use the values to update `DELTA_ENDSTOP_ADJ`,
and repeat until the error was satisfactorily small.

I *could* do some math (probably 3D trigonometry, sounds like fun)
to calculate exactly how to update `DELTA_ENDSTOP_ADJ` perfectly
and converge as quickly as possible,
but it was a lot easier to just update the variable with the raw values from the probe.
Doing it this way still converges, which is all that matters,
and since it was automatic now I didn't mind waiting.

## Controlling a printer over USB with Python

Every 3D printer I know of reads the 3D print
from an SD card / USB stick or over USB.
Many support both.
This is the premise behind [Octoprint](https://octoprint.org/) and alternatives:
rather than fiddle with SD cards, plug a computer into the printer with USB
and print remotely and without the hassle!

This was my plan: I would write some Python code to interface with my 3D printer,
send it `G30` commands, perform calculations on the results,
and update `DELTA_ENDSTOP_ADJ` with `M666`.
I had used [`pySerial`](https://pyserial.readthedocs.io/en/latest/pyserial.html) before many times, so it was the obvious choice.

Obviously, programs like [Octoprint](https://octoprint.org/) and [Pronterface](https://www.pronterface.com/)
access a printer in this way to manage prints or as a control dashboard.
But I hadn't seen any small, one-off calibration scripts like this before.
Ok, I know [one example](https://matoumakes.net/i-taught-my-3d-printer-to-play-chess.html), but I mean programs that make it a better 3D printer!

This was an experiment, so the code is hacky and undocumented.
As I said, I wasted my time writing this,
so I never polished it.
I don't recommend anybody actually use it.
However, I don't think this would be a very good programming blog post if it
didn't include any programming, so here it is.

The main function is shown below.
It sends the `G30` command near each of the towers and yields the result (it's a generator function).
```python
def probe_xyz():
    """
    Probe near each of the three towers 70mm from the center of the bed
    """
    def get_probe_result():
        """
        Protocol is very busy, so wait until we see the line we care about
        """
        while True:
            line = readline()
            if line.startswith('Bed'):
                z = float(line.split()[-1])
                return z

    # Radius of probe points
    r = 70

    # I determined that the towers were at 0 degrees, 120 degrees, and 240 degrees
    for angle in (240, 120, 0):
        x = r * math.sin(math.radians(angle))
        y = r * math.cos(math.radians(angle))

        # Probe a single point
        ser.write(f'G30 X{x} Y{y}\n'.encode())
        z = get_probe_result()
        wait_for_ok()

        yield z
```

There is also a function to get the current value of `DELTA_ENDSTOP_ADJ` using the [`M503` G-Code (Report Settings)](https://marlinfw.org/docs/gcode/M503.html).
This is needed because we want to change `DELTA_ENDSTOP_ADJ` relative to its existing value.
When we converge, the error gets smaller, but that doesn't mean we should undo the convergence by setting the adjustment to this small error.
Rather, we should update the adjustment by adding this small error.
```python
def get_endstop_adj():
    """
    Get the current value of DELTA_ENDSTOP_ADJ using M503 command
    """

    ser.write(b'M503\n')

    while True:
        # Protocol is very busy, so wait until we see the line we care about
        line = readline()
        if line.startswith('echo:  M666'):
            line = line.replace('echo:  M666', '').strip().split()
            return [float(s[1:]) for s in line]
```

Finally, tying everything together, we have this code.
I fist add the probed values to the existing values.
As it converges, the probed values (errors) will get smaller and the adjustment will get bigger.
Then I normalize so all numbers are less than zero.
Finally, I send the new values to the printer using `M666`.
After a dozen or so iterations,
the errors were quite small and I stopped it.

```python
# Add the existing endstop adjustment with the current errors from the probe
new_endstop_adj = np.array(list(probe_xyz())) + np.array(get_endstop_adj())
wait_for_ok()

# Normalize (all values must be <=0)
new_endstop_adj -= max(new_endstop_adj)

# Send new DELTA_ENDSTOP_ADJ to printer
x, y, z = new_endstop_adj
ser.write(f'M666 X{x} Y{y} Z{z}\n'.encode())
wait_for_ok()
```

This is great, right?
I was very proud at this point.
I was even planning to expand this into an Octoprint plugin.

## Why was this a waste of my time?

I just said it worked and the errors converged to zero,
so why did I say it was a waste of my time?
Well, turns out there's a little [G-Code `M33`](https://marlinfw.org/docs/gcode/G033.html).
What does it do?
It's called "Delta Auto Calibration".
It calibrates a delta, automatically.
It's not limited to `DELTA_ENDSTOP_ADJ`, this instruction calibrates almost every parameter about a delta.
The delta height (which determines how squished the first layer will be),
the delta radius,
and the delta tower angles.
The height is easy enough to calibrate with paper
(or the lazier approach of adjusting it on-the-fly during the first layer until it looks just squished enough),
but the last two are trickier without a probe.
In short, there was already a better calibration routine backed into my printer's firmware.

Why didn't I know about this?
Maybe because I had just gotten my probe.
Maybe my Google-fu was off that day.
I think I knew it existed, or had heard of it,
but I didn't know exactly what it did or how it worked.
Maybe it got mixed up in my mind with [`G29 - Bed Leveling`](https://marlinfw.org/docs/gcode/G029.html).

## Conclusions

Was this really a waste of my time?
Well, I wasted an entire Friday night coding something that was already better-implemented in my firmware,
so yes.
But at the same time, I learned a lot while doing this,
and I had fun.
I have a better idea of what's possible to do while controlling a printer from Python,
and I am keeping an eye out for another instance where this could be useful.
Octoprint is great, but I think it would be really neat to have a collection of little scripts that each do some tiny task on your printer.

## Appendix: The full code

I cleaned up the code a tiny bit of the blog post,
but it was still thrown together pretty quickly with the mindset that it was a prototype and an experiment.
I don't recommend that anyone run this on their printer.
Use it at your own risk.
With that, here is the full code.
```python
import time
import serial
import math
import numpy as np


def _readline():
    while True:
        c = ser.read()
        if c == b'\n':
            return
        yield c.decode()


def readline():
    return ''.join(_readline())


def wait_for_ok():
    while True:
        line = readline()
        if line == 'ok':
            return


def get_endstop_adj():
    """
    Get the current value of DELTA_ENDSTOP_ADJ using M503 command
    """

    ser.write(b'M503\n')

    while True:
        # Protocol is very busy, so wait until we see the line we care about
        line = readline()
        if line.startswith('echo:  M666'):
            line = line.replace('echo:  M666', '').strip().split()
            return [float(s[1:]) for s in line]


def probe_xyz():
    """
    Probe near each of the three towers 70mm from the center of the bed
    """
    def get_probe_result():
        """
        Protocol is very busy, so wait until we see the line we care about
        """
        while True:
            line = readline()
            if line.startswith('Bed'):
                z = float(line.split()[-1])
                return z

    # Radius of probe points
    r = 70

    # I determined that the towers were at 0 degrees, 120 degrees, and 240 degrees
    for angle in (240, 120, 0):
        x = r * math.sin(math.radians(angle))
        y = r * math.cos(math.radians(angle))

        # Probe a single point
        ser.write(f'G30 X{x} Y{y}\n'.encode())
        z = get_probe_result()
        wait_for_ok()

        yield z


ser = serial.Serial('/dev/ttyACM0', baudrate=250000)
time.sleep(1)

ser.write(b'G28\n')
wait_for_ok()


# Add the existing endstop adjustment with the current errors from the probe
new_endstop_adj = np.array(list(probe_xyz())) + np.array(get_endstop_adj())
wait_for_ok()

# Normalize (all values must be <=0)
new_endstop_adj -= max(new_endstop_adj)

# Send new DELTA_ENDSTOP_ADJ to printer
x, y, z = new_endstop_adj
ser.write(f'M666 X{x} Y{y} Z{z}\n'.encode())
wait_for_ok()
```
