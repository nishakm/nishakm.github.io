---
title: BlinkyBracelet - Troubleshooting LED "leakage"
desc: Where is that current coming from?
---

### The Issue

Previously on BlinkyBracelet: I left the project with a strange issue where lighting up certain LEDs also lit up other LEDs but to a lesser degree. Since I used scotch
tape to glue on the LEDs to the copper tape and the connections were punctured into the copper tape and paper, I could not be sure whether the issue was the design or the
method of prototyping.
<iframe width="560" height="315" src="https://www.youtube.com/embed/qtgl8-9lTNA" frameborder="0" allowfullscreen></iframe>

Since then, I was able to purchase a small soldering iron with some accessories. After a couple of attempts I got the hang of soldering on LEDs to copper tape.

### Breaking it down

First thing's first - break down the whole circuit into components. I make a charlieplexed circuit with 6 LEDs in 2 rows of 3 LEDs. Pins 6, 5 and 3 are PWM pins on the Arduino Uno.

<img src="{{ site.baseurl }}/assets/img/3x2_drawing.png">

Here is what that looks like on paper with copper tape and the LEDs soldered on. The source code to drive the LEDs is [here](https://github.com/nishakm/blinkybracelet/blob/master/sketch_blinkybracelet_3x2/sketch_blinkybracelet_3x2.ino)

<img src="{{ site.baseurl }}/assets/img/pesky_led.png">

So now I have a smaller set of LEDs that I can debug. Even with the small set that pesky lame LEDs issue pops up.

### Where is the current going?

Breaking it down made me realize immediately that this had nothing to do with current leakage or breaking down the LED but a timing issue with keeping multiple LEDs on in
a charlieplexed circuit.

One more time - because all of the LEDs are tied together and none of them have a common ground, the only way to turn on a specific set of LEDs was to turn each of the
LEDs in the set on and off in rapid succession. No human (or camera) can see the flickering so it works out. This is called "persistance of vision".
So essentially the "leakage" on the LEDs came from a small overlap between the states of the pins (OK, there is some actual current leakage going on but I'll get to that later). Here is a diagram (again, hand-drawn, because it's quick) that
demonstrates this.

<img src="{{ site.baseurl }}/assets/img/3x2_timing_drawing.png">

At some point the pin 3 was high and stayed high while pin 6 went to high impedence. If it had gone low then LED 6 would have gone completely on. The High Impedence is not exactly "open" and the state changes are not instantaneous so I would guess this is the reason for my mischevious LEDs turning on when they were not supposed to.

The fix was to reset them to high impedence after they completed a round of blinking.

```c
void reset2()
{
  pinMode(3, INPUT);
  pinMode(5, INPUT);
  pinMode(6, INPUT);
}
```
And then:

```c
  // set pinmode
  reset2();
  pinMode(leds[ledNo].pinHigh, OUTPUT);
  pinMode(leds[ledNo].pinLow, OUTPUT);
  pinMode(leds[ledNo].pinHiImp, INPUT);
  // turn it on
  digitalWrite(leds[ledNo].pinLow, LOW);
  digitalWrite(leds[ledNo].pinHigh, HIGH);
  delay(1);
```

And presto! No more sneaky LEDs! I earned my cookie today.

<iframe width="560" height="315" src="https://www.youtube.com/embed/2lY1nau_yNg" frameborder="0" allowfullscreen></iframe>
