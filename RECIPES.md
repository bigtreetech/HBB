# HBB recipes

Patterns that go beyond one-button-one-command. Everything here is stock
Klipper — no plugins — and each recipe is self-contained, so take only the ones
you want.

All of it assumes the `!` pin inversion from `sample-bigtreetech-hbb.cfg`. If
your pins have no `!`, every `press_gcode` below will fire when you *let go*
instead. Check first:

```
QUERY_BUTTON BUTTON=key1
```

With nothing touched that must report `RELEASED`. If it says `PRESSED`, add the
`!` — otherwise nothing below behaves the way it reads.

---

## 1. LEDs that follow printer state

The sample lights an LED when a button is pressed and never clears it. More
useful is an LED that reports what the printer is *doing* — it then lights when
a temperature is set from your slicer or web UI, and clears when a print ends,
neither of which involve the button at all.

One `delayed_gcode` re-arming itself drives all of them:

```ini
[delayed_gcode hbb_status_leds]
initial_duration: 5.
gcode:
    # key4 green while the machine is homed
    {% if "xyz" in printer.toolhead.homed_axes %}
        SET_LED LED=HBB_LED RED=0 GREEN=0.5 BLUE=0 INDEX=4
    {% else %}
        SET_LED LED=HBB_LED RED=0 GREEN=0 BLUE=0 INDEX=4
    {% endif %}

    # key6 red whenever the hotend has a target
    {% if printer.extruder.target > 0 %}
        SET_LED LED=HBB_LED RED=1 GREEN=0 BLUE=0 INDEX=6
    {% else %}
        SET_LED LED=HBB_LED RED=0 GREEN=0 BLUE=0 INDEX=6
    {% endif %}

    # key7 red whenever the bed has a target
    {% if printer.heater_bed.target > 0 %}
        SET_LED LED=HBB_LED RED=1 GREEN=0 BLUE=0 INDEX=7
    {% else %}
        SET_LED LED=HBB_LED RED=0 GREEN=0 BLUE=0 INDEX=7
    {% endif %}

    UPDATE_DELAYED_GCODE ID=hbb_status_leds DURATION=5
```

**Use one loop, not one per LED.** Every `delayed_gcode` is a timer on Klipper's
reactor, and the work runs where motion planning runs.

**Never read `printer.configfile.settings` in a loop like this.** It forces
Klipper to build the entire parsed config for the template — around 70ms of
blocked reactor per read on a Pi. Three loops each doing it three times every
five seconds was enough to cause `Timer too close` MCU shutdowns at print start,
where the host was already busy. If you want values from your config, read them
**once** at startup into macro variables:

```ini
[gcode_macro _HBB_VARS]
variable_init_r: 0.0
variable_init_g: 0.0
variable_init_b: 0.0
gcode:

[delayed_gcode _hbb_cache_colours]
initial_duration: 1.
gcode:
    {% set n = printer.configfile.settings['neopixel hbb_led'] %}
    SET_GCODE_VARIABLE MACRO=_HBB_VARS VARIABLE=init_r VALUE={n.initial_red}
    SET_GCODE_VARIABLE MACRO=_HBB_VARS VARIABLE=init_g VALUE={n.initial_green}
    SET_GCODE_VARIABLE MACRO=_HBB_VARS VARIABLE=init_b VALUE={n.initial_blue}
```

Then the status loop uses `printer['gcode_macro _HBB_VARS'].init_r` and costs
nothing. Reading live *status* (`printer.extruder.target`, `homed_axes`) is
cheap and fine in a loop — it is only `configfile.settings` that is expensive.

---

## 2. Home / motors-off on one button

A button that means "get the machine ready", and the same button means "let it
go" once it is. State comes from the printer, so it is right even if you homed
from the web UI.

```ini
[gcode_button key4]
pin: !HBB:gpio19
press_gcode:
    _BUTTON_HOME
release_gcode:

[gcode_macro _BUTTON_HOME]
gcode:
    {% if "xyz" not in printer.toolhead.homed_axes %}
        G90
        G28
        # add QUAD_GANTRY_LEVEL / Z_TILT_ADJUST / a preheat here if you want
    {% else %}
        M84
    {% endif %}
    # repaint the status LEDs now rather than waiting up to 5s for the loop
    UPDATE_DELAYED_GCODE ID=hbb_status_leds DURATION=1
```

The `UPDATE_DELAYED_GCODE` at the end is worth having: without it the LED lags
the action by up to a full loop period, which reads as the button not working.

---

## 3. Long press

`[gcode_button]` gives you two edges and no hold detection, but a
`delayed_gcode` armed on press and cancelled on release is a hold timer:

```ini
[gcode_button key6]
pin: !HBB:gpio13
press_gcode:
    _HBB_CYCLE_NOZZLE
    UPDATE_DELAYED_GCODE ID=_hbb_hold_nozzle DURATION=0.75
release_gcode:
    UPDATE_DELAYED_GCODE ID=_hbb_hold_nozzle DURATION=0

[delayed_gcode _hbb_hold_nozzle]
gcode:
    M104 S0
    UPDATE_DELAYED_GCODE ID=hbb_status_leds DURATION=1
```

`DURATION=0` cancels a pending `delayed_gcode`, so a quick tap never lets the
timer fire. Hold past 0.75s and it fires **while you are still holding**, which
is better feedback than waiting for release — you feel where the line is.

**Do not share a flag between the press, the release and the timer.** The
obvious design is for the timer to set `held = 1` and the release to skip its
action when it sees it. That races, because `gcode_button` renders the entire
template *before* running any of it:

```python
commands = template.render()      # your {% if held %} is decided HERE
self.gcode.run_script(commands)   # and only then does anything run
```

So the release picks its branch against whatever the flag was at that instant,
while the writes to it come from other `run_script` calls going through a
mutex-guarded queue on separate reactor callbacks. Near the threshold it is a
coin flip, and a stale flag from a previous hold makes the *next* tap do
nothing.

Acting on press and making the release a bare cancel removes the shared state
entirely. The cost is that a long press performs the short-press action first —
here it steps to the next preset before switching off. Visible, harmless, and
worth it for something that behaves the same way every time.

---

## 4. Cycling through preset temperatures

One button that walks a ladder and wraps back to off:

```
off -> 200 -> 220 -> 240 -> 260 -> off
```

```ini
[gcode_macro _HBB_VARS]
variable_nozzle_steps: [200, 220, 240, 260]
variable_bed_steps: [60, 80, 100]
gcode:

[gcode_macro _HBB_CYCLE_NOZZLE]
gcode:
    {% set steps = printer['gcode_macro _HBB_VARS'].nozzle_steps %}
    {% set cur = printer.extruder.target|float %}
    {% set ns = namespace(next=0.0, found=False) %}
    {% for s in steps %}
        {% if not ns.found and cur < (s|float - 0.5) %}
            {% set ns.next = s|float %}
            {% set ns.found = True %}
        {% endif %}
    {% endfor %}
    M104 S{ns.next}
    UPDATE_DELAYED_GCODE ID=hbb_status_leds DURATION=1
```

Three things make this behave well:

**It compares against the live target, not a remembered position in the list.**
So a temperature that came from somewhere else does the sensible thing — a
slicer sitting at 230 steps to 240, not back to the start of the ladder.

**`ns.next` defaults to 0.0**, so "nothing in the list is above the current
target" *is* the off case. The wrap needs no special branch, and a target above
the top of the ladder also lands on off rather than getting stuck.

**The `namespace` is required, not stylistic.** Jinja gives loop bodies their
own scope, so a plain `{% set %}` inside the `for` is discarded each iteration.

The `- 0.5` tolerance means a target sitting exactly on a preset steps *past* it
instead of re-selecting itself.

For the bed, swap `nozzle_steps` for `bed_steps`, `printer.extruder` for
`printer.heater_bed`, and `M104` for `M140`. For a `[heater_generic]` — a
chamber or drybox heater — use
`printer['heater_generic drybox'].target` and
`SET_HEATER_TEMPERATURE HEATER=drybox TARGET={ns.next}`, since `M104` and `M140`
only address the hotend and bed.

---

## 5. Putting 3 and 4 together

`key6` then gives you:

* **tap** — step to the next preset
* **hold** — straight to off
* **LED** — red whenever the hotend has a target, however it was set

which is most of what a preheat button wants to be.

---

## Notes

**Check the edge first.** `QUERY_BUTTON BUTTON=keyN` with nothing pressed should
say `RELEASED`. If it does not, add `!` to the pin.

**Nothing in a `gcode_button` template is free.** It runs on the reactor
alongside motion planning. Keep the templates short and push anything
substantial into a macro.

**Errors in button scripts are silent.** `gcode_button` catches every exception
and only writes it to `klippy.log`, so a broken template looks exactly like a
dead button. `grep "Script running error" ~/printer_data/logs/klippy.log` when
one stops responding.
