+++
title = "Designing My Own Extruder Was a Horrible Idea"
date = 2021-12-09
[extra]
header_image = "/custom-extruder/poster.jpg"
[taxonomies]
tags= ["3d_printing"]
+++

I recently designed my own custom extruder for my printer,
and have since thrown it out and reinstalled the stock one.
But why would I want a custom extruder in the first place?

<!-- more -->

Well, my AliExpress delta printer doesn't have a lot of space above the effector
for a fan.
I had a small fan attached to the heat sync, but every time the print head would go into the corner,
one of the delta rods would block the fan blades and it would make an awful grinding sound.
After upgrading to silent [Trinamic](https://www.trinamic.com/products/integrated-circuits/details/tmc2209-la/) stepper drivers,
the fan noise became very noticeable and annoying,
so I wanted to upgrade the cheap little fan with a bigger quieter Noctua.

The fan is very important to prevent heat creep.
Once or twice, I got very annoyed at the fan noise and ripped it off,
only to have a clog a few seconds later.
Heat creep is a phenomenon where heat creeps (who would have guessed)
from the hot plastic-melty bits up through the heat break to where the filament is supposed to be hard and rigid.
When the filament becomes soft too soon, it can't be pushed through.

Long story short, I designed my own extruder
with the heat sink and fan *below* the effector where there is space.
I also slapped a BLTocuh bed probe on the side.
No more paper wiggling for this guy!

![CAD of my custom extruder](/custom-extruder/cad.png)

Very proud of myself,
I printed out all the parts and installed them to my machine.
I proceeded to run mesh bed levelling to test out my fancy new BLTocuh.
The calibration completed successfully, but the first layer was awful!
I ran it again just to be sure, but still a horrible first layer.
What gives? Everybody online is raving about this sensor.

I touch on this in [my previous post](/aluminum-foil-bed-probe),
but to probe accurately with a delta printer,
it either has to be constructed absolutely perfectly,
or the sensor has to be aligned with the nozzle.
The reason is that there is often a small amount of tilt in the effector as it moves from side to side.
There isn't a lot of adjustment in my printer just because of the way it's designed,
so I don't think I'll be able to make it "absolutely perfect".
I spent an entire weekend measuring things on it, but there isn't really anything I can adjust.
Furthermore, what I had done by putting the nozzle and probe so far below the effector was multiply the error caused by this small angle.

I ripped out the BLTouch and enabled the `NOZZLE_AS_PROBE` option.
I don't have one of those snazzy [strain gauge effectors](https://www.duet3d.com/DeltaSmartEffector),
but this nozzle as probe trick is pretty neat.
It completes the circuit between your metal nozzle and metal bed as a probe.
This was ok; I at least had a level first layer, but I had other problems.

![Nozzle Distance to Effector With Custom (Left) and Stock (Right) Extruders](/custom-extruder/dimensions.png)

Effector tilt does not only complicate bed probing,
it also degrades dimensional accuracy.
The image above shows a representation of the distance between the nozzle and the effector
of my custom extruder (left) and the stock design (right).
Let's assume there is 2 degree of effector tilt.
The cosine error (vertical error) is negligible for both cases.
The sine error (horizontal error) however is 0.73mm for the stock extruder
and a whopping 2.31 for my custom extruder.
Go figure that swinging the nozzle around 60mm from the centre of rotation
would make prints suffer.
It was so bad that printing a square would produce a parallelogram.
It was visible by eye, and the diagonals were several millimeters different.

I tried everything to compensate for this effect.
I found [a calibration piece and accompanying write-up on Thingievers](https://www.thingiverse.com/thing:1274733).
This teaches you how to calibrate the delta rod length and rod trim (adjustable through the `M665` command)
and compensate for the parallelograms.
I have 100 of these test prints now.

This was ok for a while, but it's a hack.
I wasn't entirely convinced that this compensation was working properly and wouldn't have adverse effects on other parts of the print.
Therefore, I scrapped my design and remounted the nozzle as close to the effector as I could.
