# nora-hal-dspic33ck-gpio

**NORA-HAL** — *Native On-chip Resource Assistant*

Small, readable GPIO and PPS HAL for Microchip dsPIC33CK devices — part of
**NORA-HAL**, a HAL family whose public API is namespaced `nora_*` / `NORA_*`.

This project is intended as a compact alternative to large generated driver code.
The goal is not to hide everything behind a framework, but to provide a simple GPIO
layer that is easy to read, test, modify, and adapt.

## Naming

The public API is `nora_*` / `NORA_*`.

The chip name survives in exactly two places, both deliberate:

* **Implementation file names** carry a backend tag: `nora_gpio_dspic33ck.c` is the
  dsPIC33CK backend of the processor-neutral `nora_gpio.h`. A second processor would
  add `nora_gpio_<tag>.c` beside it, not a second header.
* **Backend-private identifiers** inside those files (the register helpers in
  `nora_gpio_dspic33ck_reg.h`, the per-port table, statics), which no caller sees.

The tag is `_dspic33ck`, the device family this backend actually drives — a
different silicon family from dsPIC33AK (dsPIC33**C** vs dsPIC33**A**), and never
abbreviated to `_dspic33c`.

### Relationship to the dsPIC33AK HAL of the same name

[nora-hal-dspic33ak-gpio](https://github.com/sulaolab/nora-hal-dspic33ak-gpio) is
the same API shape for the dsPIC33AK family — deliberately so, and the packed-pin
and RP-first call surfaces match. What differs:

| | dsPIC33CK (here) | dsPIC33AK |
|---|---|---|
| GPIO SFR width | **16-bit** (`nora_gpio_dspic33ck_reg.h` uses `volatile uint16_t *`) | 32-bit |
| CN event registers | 16-bit, and the CN interrupt lines sit in different `IEC`/`IFS` words | 32-bit |
| PPS signal set | CK's own: SPI1..4, UART1..3, CLC1..10, PWM1..8, CMP1..8, REFO1/2, REFI1/2, CAN1/2, ICM1..9, INT1..4 | the AK set (no CMP/REFO group; PWM and CLC ranges differ) |
| RP → register slot | **not affine on every part** — see `nora_pps_find_output_rp()` below | — |
| `nora_gpio_table` | present | present |
| `nora_pps_find_output_rp()` | present here | **not in the published AK HAL** |

`nora_gpio_table.{h,c}` is available in both public HALs. The dsPIC33AK and
dsPIC33CK fleets are **not** otherwise symmetric, and nothing here should be read
as a claim that they are.

## Status

Validation target:

- Devices: dsPIC33CK64MC105 (EV88G73A Curiosity Nano) and dsPIC33CK256MP508 (DM330030)
- Compiler: XC-DSC v3.31.01
- DFP: dsPIC33CK-MC_DFP 1.10.386 / dsPIC33CK-MP_DFP 1.15.423 or compatible

The per-port register table is built from `#if defined(...)` presence tests on the
device header, so it adapts to whichever ports the selected device defines, without
device-name conditionals.

**How to read the evidence below.** These HALs are built for evaluation, FAE demos and
architecture experiments, so exhaustive per-function coverage was never the goal — there is
no unit-test suite, and what exists is integration testing on real hardware. Three tiers,
used across the seven sibling repositories: **integration-verified** (it ran as part of the
working system, and something observable would have broken if it had not),
**hardware-observed, not a matrix** (it worked in the configuration actually run; other
combinations are untried rather than known-good), and **compiled, not executed**.

**Exercised on hardware (EV88G73A, 2026-08-11):**

* LED output — a tick-driven blink, observed against a stopwatch
* button input (SW0) with an internal pull
* WM8904 codec control pins and I²C pin pre-configuration
* SPI/I2S/TDM pin pre-configuration and PPS routing for a live audio stream
  (BCLK / FS / data in and out), soaked with `miss = 0` over >137,000 audio blocks
* `nora_pps_find_output_rp()` — the frame-sync generator in the SPI/I2S/TDM HAL asks
  it "which pad is this signal on"
* the one-shot `nora_gpio_config()` apply order, and `nora_gpio_table_apply()` as
  the board's pin stage

**Builds clean but unexecuted:**

* `nora_gpio_event_dspic33ck.c` (the CN event layer) — it is **excluded from the
  EV88G73A configuration**, and included only in the `CK256MP508_DM330030`
  configuration, which is compile-only. So the CN layer
  compiles on every build of that configuration and **has not run on silicon**.
  Its declarations carry the same warning.
* dsPIC33CK256MP508 as a whole, for the same reason.

## Design policy

This HAL is intentionally small.

* A pin is addressed either by its Remappable-Pin number (RPn) through the
  **RP-first API** (preferred for normal board and application code) or by a packed
  `(port, bit)` handle through the **core packed-pin API** (for non-RP pins,
  low-level GPIO, or HAL internals). Both reach the same register implementation;
  the RP-first functions are thin wrappers that convert RPn to a packed pin and
  delegate to the core.
* No XC-DSC / DFP bitfield structures (`LATxbits` / `TRISxbits` / ...) are exposed
  in the public API.
* Device-specific register symbols are isolated in a small per-port pointer table,
  built from presence tests rather than part numbers.
* The core GPIO layer owns only the GPIO attribute/data registers (`ANSEL` / `TRIS` /
  `LAT` / `PORT` / `CNPU` / `CNPD` / `ODC`). PPS signal routing is a separate,
  optional companion module (`nora_pps.h` / `nora_pps_dspic33ck.c`); the board layer
  owns the policy — which signal maps to which RP pin.
* The core GPIO layer does not own interrupt vectors. The optional CN event layer
  only dispatches registered GPIO events when the application calls it from an
  app-owned vector.
* The accessors are plain read-modify-write and do **not** disable interrupts; if a
  port is updated from both main-line code and an ISR, the caller provides the
  mutual exclusion.

## Scope

In scope:

* Direction, pull, analog/digital, open-drain configuration
* Output write / set / clear / toggle
* Input read and output-latch read-back
* RP-first addressing (RPn) as a thin adapter over the packed-pin core
* A board's whole fixed pin stage as one table (`nora_gpio_table.h`)
* Optional PPS (peripheral pin select) routing, including a reverse lookup
* Optional Change Notification (CN) event attach/detach/dispatch
* Any port present on the device

Out of scope (not handled here):

* HAL-owned interrupt vectors
* Debounce or event semantics beyond the CN event layer
* Atomic set/clear via dedicated SFRs (accessors are read-modify-write)
* RTOS locking / cross-context mutual exclusion
* Package-level pin validity — this HAL checks whether a *port* exists in the
  device header, not whether a given package exposes that pin. That stays with the
  board layer.

## Files

Headers are processor-neutral and untagged; each `.c` is a backend and carries the
`_dspic33ck` tag.

```text
src/
  nora_gpio.h                     core GPIO API (attributes + data) + RP-first adapter
  nora_gpio_dspic33ck.c           dsPIC33CK backend
  nora_gpio_dspic33ck_reg.h       dsPIC33CK register layer (16-bit SFR helpers)
  nora_pps.h                      optional PPS (peripheral pin select) routing
  nora_pps_dspic33ck.c            dsPIC33CK backend
  nora_gpio_table.h               optional board pin stage as data
  nora_gpio_table_dspic33ck.c     dsPIC33CK backend
  nora_gpio_event.h               optional Change Notification (CN) event layer
  nora_gpio_event_dspic33ck.c     dsPIC33CK backend — builds clean, unexecuted
```

The optional companions are compiled only when their feature is used: compile
`nora_gpio_event_dspic33ck.c` only when CN event support is needed,
`nora_pps_dspic33ck.c` only when the board routes peripherals through PPS, and
`nora_gpio_table_dspic33ck.c` only when a board describes its pins as a table.

## Pin addressing

Two addressing styles are supported. Use RP-first for PPS-capable board and
application pins; use packed-pin for non-RP pins or low-level core usage.

**RP-first (preferred for board/application code with PPS):** a pin is identified by
its Remappable-Pin number `RPn` — the same number the PPS map uses — so the GPIO
call and the PPS route refer to the pin identically:

```c
#define BOARD_UART1_TX_RP  ((nora_gpio_rp_t)58u)   /* U1TX -> RP58 (RC10) */

nora_gpio_rp_config_digital_output(BOARD_UART1_TX_RP, true);
nora_pps_route_output(NORA_PPS_OUTPUT_U1TX, BOARD_UART1_TX_RP);
```

**Packed-pin (core API — for non-RP pins or HAL internals):** a pin is a packed
number `(port << 4) | bit`. Always build it with `NORA_GPIO_PIN(port, bit)` and give
it a board-level name:

```c
#include "nora_gpio.h"

#define BOARD_LED0      NORA_GPIO_PIN(NORA_GPIO_PORT_D, 10u)   /* EV88G73A LED0 */
#define BOARD_SW0       NORA_GPIO_PIN(NORA_GPIO_PORT_D, 13u)   /* EV88G73A SW0  */
```

Port codes are `NORA_GPIO_PORT_A` .. `NORA_GPIO_PORT_H`; `bit` is `0..15`. The enum
spans the family's widest port set — a port that the selected device does not define
simply has no row in the register table, and calls against it are refused (see
[API summary](#api-summary)). Values outside the ranges above are masked by the
macro and should not be used.

## Basic usage

One-shot configuration (recommended) applies attributes in a glitch-aware order:

```c
const nora_gpio_config_t led_cfg = {
    .dir          = NORA_GPIO_DIR_OUTPUT,
    .pull         = NORA_GPIO_PULL_NONE,
    .analog       = false,
    .open_drain   = false,
    .initial_high = false,   /* LAT seeded before the pin becomes an output */
};
(void)nora_gpio_config(BOARD_LED0, &led_cfg);

nora_gpio_set(BOARD_LED0);      /* drive high            */
nora_gpio_clear(BOARD_LED0);    /* drive low             */
nora_gpio_toggle(BOARD_LED0);   /* flip the output latch */
```

Reading an input:

```c
const nora_gpio_config_t btn_cfg = {
    .dir = NORA_GPIO_DIR_INPUT, .pull = NORA_GPIO_PULL_UP,
    .analog = false, .open_drain = false, .initial_high = false,
};
(void)nora_gpio_config(BOARD_SW0, &btn_cfg);

/* read() returns a 3-state level (ERROR / LOW / HIGH) -- not a bool. Handle
 * NORA_GPIO_LEVEL_ERROR first, then compare against LOW / HIGH. */
bool pressed = (nora_gpio_read(BOARD_SW0) == NORA_GPIO_LEVEL_LOW);  /* active-low */
```

Individual attribute setters are also available when one-shot config is not wanted:

```c
(void)nora_gpio_set_analog(BOARD_LED0, false);
(void)nora_gpio_set_pull(BOARD_LED0, NORA_GPIO_PULL_NONE);
(void)nora_gpio_set_open_drain(BOARD_LED0, false);
(void)nora_gpio_set_direction(BOARD_LED0, NORA_GPIO_DIR_OUTPUT);
```

## A board's pin stage as a table

A board's user-I/O stage is a list — this pin an output starting low, that one an
input with a pull-up, this one analog. Written as straight-line code it becomes one
`if (!...) return false;` per pin, and the *list*, which is the actual board fact,
disappears into the control flow that walks it. `nora_gpio_table.h` makes the list
the data and shares the walk:

```c
#include "nora_gpio_table.h"

static const nora_gpio_table_entry_t board_pins[] = {
    { BOARD_LED0, &nora_gpio_cfg_output_low },
    { BOARD_SW0,  &nora_gpio_cfg_input_pullup },
    /* ... */
};

if (!nora_gpio_table_apply(board_pins, (uint8_t)(sizeof board_pins / sizeof board_pins[0]))) {
    /* one entry was refused; the walk stopped there */
}
```

**Entries are independent.** The walk is top to bottom because it is a loop, but no
board's table may rely on that: each entry states one pin completely (direction,
pull, analog, open-drain, initial level), so any order gives the same result. If a
new board seems to need pin B configured after pin A, the pin descriptions are
incomplete — fix the description rather than relying on the sequence.

The interface is deliberately thin: `nora_gpio_table_apply()` reports pass/fail and
**not** which entry refused. A table makes that nearly free to compute, but the
straight-line code it replaces did not report it either, no caller has asked, and
this links into a 64 KB part with section GC off — so unused reporting is not free.

## PPS routing (peripheral pin select)

`nora_pps.h` / `nora_pps_dspic33ck.c` is an optional companion module that maps a
peripheral signal to or from a Remappable-Pin (RPn). The board layer owns the policy
and uses the *same* RPn for the GPIO attribute call and the PPS route:

```c
nora_gpio_rp_config_digital_output(BOARD_UART1_TX_RP, true);   /* GPIO first */
nora_pps_route_output(NORA_PPS_OUTPUT_U1TX, BOARD_UART1_TX_RP);

nora_gpio_rp_config_digital_input(BOARD_UART1_RX_RP);
nora_pps_route_input(NORA_PPS_INPUT_U1RX, BOARD_UART1_RX_RP);
```

API:

* `nora_pps_route_output(signal, rp)` — drive a peripheral output onto RPn. Returns
  `false` (routing nothing) if the signal or the RP pin is not defined on the
  selected device.
* `nora_pps_route_input(signal, rp)` — feed a peripheral input from RPn. Rejects an
  `rp` that is not a physical pin on the device (returns `false` before writing).
* `nora_pps_unlock()` / `nora_pps_lock()` — the `RPCON.IOLOCK` gate. The `route_*`
  functions open and close it themselves; these are exposed only for code that
  writes PPS registers directly.
* `nora_pinmux_route_output(signal, rp, initial_high)` / `nora_pinmux_route_input(signal, rp)`
  — the GPIO digital configuration **and** the route in one call, in the
  glitch-aware order. They apply the GPIO step first and return `false` without
  routing anything if it fails; they add no register access of their own.
* `nora_pps_find_output_rp(signal, &rp)` — **reverse lookup**: scan the device's
  output PPS registers for that signal's function code and report the first
  *physical* pin carrying it (virtual pins are skipped). Read-only, so it does not
  touch IOLOCK.

### Why the reverse lookup belongs in the HAL

It answers "where did the board route this", which a peripheral driver otherwise
cannot ask without reimplementing the `RPORx` bank layout — and that layout is the
thing most easily got wrong, because **RPn is not an affine function of the register
slot on every part**: CK64MC105's remappable pins are non-contiguous above RP61, so
slot arithmetic silently addresses unrelated SFR space. The frame-sync generator in
the SPI/I2S/TDM HAL had exactly that bug, from exactly that cause, before this call
existed. Keeping the map in one place — the flat per-RP switch this file already
maintains for writing — means there is only one copy to be right.

### The signal set, and adding to it

The enums carry the signals this codebase currently needs — SPI1..SPI4 (SS/SCK/SDO
out, SS/SCK/SDI in), UART1..UART3 (TX/RX), CLC1..CLC10 out and CLCINA..CLCINJ in,
PWM1H..PWM8H/8L and the four PWM event outputs, CMP1..CMP8, REFO1/REFO2 out,
REFI1/REFI2 in, CAN1/CAN2 (TX/RX), Input Capture ICM1..ICM9, and INT1..INT4. They
are **not** every PPS-capable peripheral the family supports.

Extend the enum, then add the matching device-guarded case — no part-number
conditionals:

1. Add an enumerator to `nora_pps_output_t` (outputs) or `nora_pps_input_t` (inputs)
   in `nora_pps.h`.
2. Add the matching `#ifdef`-guarded case in `nora_pps_dspic33ck.c`.

Device adaptation is entirely by those `#ifdef`s on the XC-DSC header macros, so a
signal or RP the selected device header does not define is simply left out of the
switch and the route call returns `false`.

## API summary

### Packed-pin core API

Configuration (glitch-aware order):

* `nora_gpio_config()` — one-shot via config struct
* `nora_gpio_config_digital_input()` — shortcut: digital input, no pull
* `nora_gpio_config_digital_output()` — shortcut: digital output, seeds LAT first
* `nora_gpio_set_direction()` / `_set_pull()` / `_set_analog()` / `_set_open_drain()`

Output:

* `nora_gpio_write()` — drive a given level
* `nora_gpio_set()` / `nora_gpio_clear()` / `nora_gpio_toggle()`

Input / read-back (return a 3-state `nora_gpio_level_t`, **not a bool**):

* `nora_gpio_read()` — pin level from `PORT`
* `nora_gpio_read_output()` — driven latch from `LAT`

### RP-first API

Thin wrappers over the packed-pin core (convert RPn → packed pin, then call the
matching core function). Preferred for board and application code that uses
PPS-capable pins.

* `nora_gpio_pin_from_rp()` / `nora_gpio_rp_from_pin()`
* `nora_gpio_rp_config()`
* `nora_gpio_rp_set_direction()` / `_set_pull()` / `_set_analog()` / `_set_open_drain()`
* `nora_gpio_rp_config_digital_input()` / `_config_digital_output()`
* `nora_gpio_rp_write()` / `_set()` / `_clear()` / `_toggle()`
* `nora_gpio_rp_read()` / `_read_output()`

### Table layer

* `nora_gpio_table_apply()` — apply every entry in order, stopping at the first
  refusal.
* `nora_gpio_cfg_output_low` / `_output_high` / `_input_pullup` / `_analog_input` —
  the four descriptions every board was otherwise writing out for itself.

### Optional event layer (builds clean, unexecuted)

* `nora_gpio_event_attach()` / `_detach()` — register/unregister one packed-pin event
* `nora_gpio_event_irq_enable()` / `_irq_disable()` — configure the CN port interrupt
  line for setup/teardown (these do clear a pending CN event)
* `nora_gpio_event_irq_is_enabled()` / `_irq_set_enabled()` — read/write only the
  enable bit, for a brief critical section that must **not** clear a pending event
* `nora_gpio_event_rp_*()` — RP-first wrappers for the above
* `nora_gpio_event_process_isr()` — app-called dispatcher; clears the handled
  per-pin `CNF` bits

`attach()` configures CN edge detection only — it does not change ANSEL, TRIS, CNPU
or CNPD. The application owns the vector:

```c
void __attribute__((__interrupt__, no_auto_psv)) _CNDInterrupt(void)   /* the pin's CN port line */
{
    nora_gpio_event_process_isr();
}
```

Setters take a packed pin from `NORA_GPIO_PIN()` (or use the RP-first wrappers) and
return `false` if the pin's port is not present on the device (no register row),
otherwise `true`. `nora_gpio_read()` / `read_output()` return a 3-state
`nora_gpio_level_t` — `NORA_GPIO_LEVEL_ERROR` (`-1`) for a pin whose port is not
present, else `..._LOW` (`0`) / `..._HIGH` (`1`). Do not use the result directly as
a bool (ERROR is truthy); compare against the named constants and handle ERROR
first.

## Glitch-aware order

`nora_gpio_config()` applies an **output** pin as: select digital, set pull and
open-drain, seed the output latch (`initial_high`), and only then switch the pin to
an output. Seeding `LAT` before flipping `TRIS` avoids driving an undefined level for
one cycle when a pin first becomes an output.

The **input** case is ordered the other way round: `TRIS` is released *first*, so a
pin that is currently a live output stops driving before its analog, pull and
open-drain attributes change underneath it. That makes a runtime output-to-input
transition safe, not just first-time configuration.

## Notes

* The GPIO SFRs are 16-bit on this family; the register layer uses `uint16_t`
  pointers to match the DFP device headers.
* This HAL checks whether a GPIO port exists in the selected device header. It does
  not check whether a specific package exposes that pin.
* This repository does not include Microchip DFP header files.
* `nora_pps.h` and `nora_gpio.h` are compile-time dependencies of
  [nora-hal-dspic33ck-spi-i2s-tdm](https://github.com/sulaolab/nora-hal-dspic33ck-spi-i2s-tdm),
  whose pin and frame-sync layers route through them.

* Sibling repositories for this family:
  [dma](https://github.com/sulaolab/nora-hal-dspic33ck-dma) ·
  [timer](https://github.com/sulaolab/nora-hal-dspic33ck-timer) ·
  [clock](https://github.com/sulaolab/nora-hal-dspic33ck-clock) ·
  [i2c](https://github.com/sulaolab/nora-hal-dspic33ck-i2c) ·
  [uart](https://github.com/sulaolab/nora-hal-dspic33ck-uart) ·
  [spi-i2s-tdm](https://github.com/sulaolab/nora-hal-dspic33ck-spi-i2s-tdm)

## License

MIT No Attribution License (MIT-0). See [LICENSE](LICENSE).

Attribution is appreciated but not required.
