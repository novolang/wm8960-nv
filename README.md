# wm8960-nv

A driver for the WM8960, a stereo audio codec with headphone drivers and
1 W class-D speaker drivers from Cirrus Logic (formerly Wolfson),
specified in the data sheet "WM8960: Stereo CODEC with 1W Stereo Class-D
Speaker Drivers and Headphone Drivers for Portable Audio Applications",
Rev 4.4, January 2019. The driver implements the `AudioCodec` trait of
[devices-nv](https://novo-lang.org/packages/devices-nv) and configures the
board's `I2s` handle of
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) for the
format it sets.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
chip, and no board package has an I2S block yet. The driver has run only
against a double of the part and a double of the I2S handle, in the host
suite. The device program below builds for the nRF52840-DK and the
nRF54LM20-DK against a stand-in handle and has not been flashed.

## Install

```sh
novo pkg add wm8960-nv
```

The device program is `examples/playback.nv`.  Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/playback.nv` brings the codec up and plays a square wave of 441 Hz
at 44.1 kHz on the headphone and speaker outputs, printing one line per step
over the board's UART:

```
codec reset
```

```novo norun:needs-pkg
use embedded_hal.{ I2s, I2sConfig, BoardI2c }
use dev_audio.{ AudioCodec, AudioFormat, AudioFraming, CodecPath, MilliDb }
use dev_bus
use wm8960

// The board's I2S block goes here; this stand-in refuses to be configured.
@value
struct NoI2s
    id: Int

impl I2s for NoI2s
    fn configure(self, c: I2sConfig) -> Bool [hw]
        false
    fn sample_rate(self) -> Int [hw]
        0
    fn enable(self) -> Bool [hw]
        false
    fn disable(self) [hw]
        pass
    fn transmit(self, var buf: Vec[i32; 256]) -> Int [hw]
        -1
    fn receive(self, var buf: Vec[i32; 256], n: Int) -> Int [hw]
        -1
    fn xruns(self) -> Int [hw]
        0

@tier(embedded)
fn main() [hw, mutate]
    bsp.board.init()
    // The codec answers at 0x1A on the first I2C bus; the board's I2S block
    // gives it MCLK at 11.2896 MHz.
    let part = wm8960.attach(BoardI2c { id: 0 }, dev_bus.wiring_i2c(0, 0x1A),
                             NoI2s { id: 0 }, wm8960.config_default(11289600))
    if not part.begin()
        hal.uart.write("no answer from the codec at 0x1A\n")
        return
    let f = AudioFormat { sample_rate: 44100, bit_depth: 16, channels: 2,
                          framing: AudioFraming.AudioPhilips, codec_is_master: false }
    if part.codec_set_format(f) and part.codec_set_power(true)
        let set = part.codec_set_gain(CodecPath.CodecOutput, MilliDb { mdb: -20000 })
        if set != None
            // After the first buffer is queued the output is opened.
            let _ = part.codec_set_mute(false)
```

The full program, which also fills and sends the buffers, is
`examples/playback.nv`. A board package that depends on this package builds
it as its `main`. `tests/device_build.sh` does so from a scratch package for
the two boards below and for the emulated board, where it runs the image and
checks that the program reports that the codec does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 20,032 B | 4,694 B | 3,764 B |
| `nrf54lm20-dk` | 21,392 B | 5,056 B | 3,764 B |

The numbers are from the build's memory report. Flash also holds the
runtime, the compiler's support library and the program's tables. The RAM
figure includes the driver's shadow of the registers, 64 words. The arena is
the runtime's heap and is reserved whether or not the program uses it.

**The I2S block is absent hardware on both boards; the I2C bus is not.**
embedded-hal-nv has a trait for an I2S interface and no board handle for
one, and neither Nordic board package drives its I2S block.  The Nordic DKs
drive the control port's I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA
P0.26 and SCL P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04),
on the pins `docs/hal/README.md` names under "The Nordic DKs' buses", so
both images carry the driver's reading path.  Before the Nordic buses landed
the two rows were 14,108 and 15,436 bytes of flash, of which the program was
2,032 and 2,358 bytes, and 3,172 bytes of RAM, because the board packages
answered for I2C as absent hardware and the build removed the code that
follows the first bus read.  The program is written against the `I2s` trait
and is given `NoI2s`, so on both images `begin` answers `false` and the
program prints `no answer from the codec at 0x1A`.  A reply from the codec
over a DK's bus is NOT VERIFIED until the driver rig carries one.  The one
argument of `attach` is what changes when a board package gives a handle.

### The effects

Every function of the driver that reaches the bus or the I2S handle declares
`[hw]`, and the functions that read or change the shadow declare `[mutate]`
as well, because the shadow is module state (see below). The driver performs
no input or output of its own. `attach` allocates one cell of the arena,
because a struct with a generic field cannot be a `@value`; the writes build
only the two-byte frames of devices-nv's register functions, in the caller's
frame. The module checks at every tier, `embedded` included.

## What it is

The WM8960 has two 24-bit sigma-delta ADCs behind input PGAs with a
microphone interface and a bias output, two 24-bit DACs behind output
mixers, a headphone driver (HP_L, HP_R) and a class-D speaker driver that
gives 1 W per channel into 8 ohm, a PLL, and an I2S, left-justified,
right-justified or DSP audio interface. It supports sample rates of 8,
11.025, 12, 16, 22.05, 24, 32, 44.1 and 48 kHz.

**The registers cannot be read.** The control port is write-only. A write is
two bytes: the register number in the top seven bits of the first byte and
the ninth data bit in its bit 0, then the low eight bits. The device address
is 0x1A. The driver therefore keeps a **shadow** of the 56 registers, nine
bits each, starting from the data sheet's reset values when it resets the
part, and sets a field by writing the shadow's other bits with it. The
volume registers' update bit is an action and is never kept in the shadow.

The shadow is module state, because `AudioCodec` gives a method a plain
`self`, which cannot change a field. The module therefore holds the state of
one WM8960 per program. `shadow_reset` and `shadow_clear` are public so a
test or a program that resets the part by other means can resynchronise it.
While the shadow is empty the driver does not guess: updates are refused
without a transaction, and the reads that come from the shadow answer `None`.

The audio clock: SYSCLK is MCLK, MCLK divided by 2, or the PLL's output. The
ADC and DAC sample rate is SYSCLK / (256 x d) with d one of 1, 1.5, 2, 3, 4,
5.5 and 6. The PLL multiplies MCLK, halved first when the ratio would be
below 6, by R = N.K with a fixed divide by 4 and a post-divide by 2, so f2 =
8 x SYSCLK:

    N = int R, 6 to 12        K = int(2^24 x (R - N))

`plan_of` chooses SYSCLK as 12.288 MHz for 8, 12, 16, 24, 32 and 48 kHz and
11.2896 MHz for 11.025, 22.05 and 44.1 kHz, and uses MCLK directly when it
is SYSCLK or twice it:

| MCLK | SYSCLK | Prescale | N | K |
|---|---|---|---|---|
| 12 MHz | 11.2896 MHz | 1 | 7 | 0x86C226 |
| 12 MHz | 12.288 MHz | 1 | 8 | 0x3126E9 |
| 24 MHz | 11.2896 MHz | 2 | 7 | 0x86C226 |
| 24 MHz | 12.288 MHz | 2 | 8 | 0x3126E9 |

The data sheet's Table 45 prints 0x3126E8 for the 12.288 MHz rows, and its
worked example prints 0x3126E9. Several other rows of the table differ from
the formula by one in the last digit. The driver computes the formula, and a
test checks all eighteen rows of the table to within one count of K. The
PLL is switched off while it is changed and given 250 ms to lock before
SYSCLK is taken from it.

The class-D clock is SYSCLK / DCLKDIV and the data sheet asks for 700 to 800
kHz, 768 kHz best. The driver chooses the divider that gives it, 16 for both
SYSCLKs (768 kHz and 705.6 kHz), and does not enable the speaker outputs
until a format has set the clock.

## What the package contains

| Module | What is in it |
|---|---|
| `wm8960` | `Wm8960Config`, `Wm8960Port`, the control word and I2C framing, the shadow, the clock plan, the named steps, the gain functions, the type `Wm8960` and its trait |

## How to choose an entry point

A program on a board builds a `Wm8960Config` with `config_default(mclk_hz)`,
which powers both outputs and the microphone path and uses zero-cross volume
changes, passes it with the board's I2C bus, the wiring and the I2S handle to
`attach`, and uses the trait on the result.

| Method | What it does |
|---|---|
| `attach(bus, w, i2s, cfg)` | the codec at `w.addr` and the handle that carries its samples; nothing is sent |
| `begin()` | the bus ready, the reset (R15), the shadow at its reset values, the zero-cross timeout |
| `codec_set_format(f)` | `AudioCodec`: configures and enables the I2S handle, then sets the interface, the PLL and the dividers |
| `codec_set_power(on)` | brings the bias, the DAC and the output drivers (and the inputs) up, with the DAC soft-muted, or down and disables the handle |
| `codec_set_mute(on)`, `codec_is_muted()` | the DAC's soft mute, read from the shadow |
| `codec_set_gain(path, g)`, `codec_gain(path)` | the output volume of the headphone and speaker drivers, or the input PGAs |
| `present()` | whether the part acknowledges its address |
| `set_dac_volume(g)`, `set_adc_volume(g)` | the digital volumes |
| `set_headphone_volume(g)`, `set_speaker_volume(g)` | one driver's volume |
| `set_mic_boost(g)`, `set_mic_bias(low)` | the input boost of 0, 13, 20 or 29 dB and the bias voltage |
| `set_speaker_boost(step)`, `set_dsp_format(master, depth, b)` | the class-D boost, and DSP mode |
| `reg(r)`, `set_reg(r, v)`, `i2s_stop()` | the shadow's value of a register, a write, and the handle's clocks stopped |

The steps are functions over `Wm8960Port`, a trait a type holding an I2C bus
implements, so a test runs them over a double of the part: `step_reset`,
`step_base`, `step_bias_up`, `step_bias_down`, `step_outputs_up`,
`step_inputs_up`, `step_power_up`, `step_power_down`, `step_interface` and
`step_clocks`.

## The power sequence

Rev 4.4 of the data sheet gives the anti-pop registers' bits and the advice
to enable and disable functions "in the correct order", and does not hold a
sequence. The order below comes from the bits' descriptions and from the
reference drivers; no board has run it.

Power up:

1. R28 (anti-pop 1) = POBCTRL, SOFT_ST, BUFDCOPEN and BUFIOEN.
2. R25 VMIDSEL = 01, a 2 x 50k divider, and 100 ms for it to ramp.
3. R25 VREF.
4. R28 = BUFIOEN alone.
5. The DAC's soft mute is set. The output mixers (R47), the DAC to mixer
   routes (R34, R37) and the DAC (R26) come on, then the headphone drivers,
   then the class-D drivers and R49 SPK_OP_EN. The class-D outputs are
   refused until a format has set the clock.
6. With the microphone path: the input PGAs, the ADCs and the microphone
   bias (R25), the PGA enables (R47), and both PGAs unmuted, the right write
   carrying the update bit.

Power down is the reverse: the DAC soft-muted, the class-D outputs off, the
drivers, the DAC, the mixers and the inputs off, then R28 as in step 1, VMID
and VREF off, and the PLL and its select off. The clock is gone, so the format
has to be set again. A power-up that follows waits 600 ms first, so the
headphone capacitors have discharged. The DAC is soft-muted after a reset and
after a power-up, and stays so until `codec_set_mute(false)`.

## The volume

The headphone and speaker volumes are 1 dB steps from -73 dB to +6 dB (codes
0x30 to 0x7F; the codes below 0x30 are the analogue mute and read as -73 dB,
the lowest step). `codec_set_gain(CodecOutput, g)` sets the headphone and
speaker pairs, and `codec_set_gain(CodecInput, g)` the input PGAs, from
-17.25 dB to +30 dB in 0.75 dB steps. The digital volumes, on top, are 0.5 dB
steps: the DAC's from -127 dB to 0 dB, the ADC's from -97 dB to +30 dB, with
code 0 the digital mute. Each pair is written as the data sheet's Figure 16
shows: the left register without its update bit, then the right with it, so
the two channels change together. With zero-cross updates a change waits for
a zero crossing, and a timeout (R23 TOEN) updates it if none comes.

The input PGA registers keep the zero-cross bit in bit 6 and the mute in bit
7, where the output volume registers keep the zero-cross bit in bit 7. A
driver that uses the output layout for the inputs mutes them.

## The rules a user needs

1. Wait for the supply, then call `begin`. The part answers its address only
   when it is powered and its I2C pins are wired; `begin` answers whether it
   acknowledged the reset.
2. Set the format before powering up. `codec_set_format` configures the I2S
   handle and enables it first, because the PLL needs MCLK running.
3. `codec_set_power(true)` leaves the DAC soft-muted and the volumes where
   they are, which is mute after a reset. Set the volume, then
   `codec_set_mute(false)`.
4. `codec_set_power(false)` loses the clock. Set the format again.
5. A master codec needs a bit clock that divides SYSCLK by 1, 1.5, 2, 3, 4,
   5.5, 6, 8, 11, 12, 16, 22, 24 or 32. 24-bit slots make 48 times the rate
   and are refused as a master.
6. One WM8960 per program. A second would share the shadow.

## What is not included

- The ALC, the limiter and the noise gate, the 3D stereo enhancement, the
  de-emphasis, the companding modes, the jack detection and the thermal
  shutdown settings: `reg` and `set_reg` reach their registers.
- The 20-bit word length, which `AudioFormat` cannot name, and the
  right-justified format.
- The OUT3 and mono output, and the capless headphone mode.
- `Identifiable`: the part has no identification register and cannot be read.
  `present` writes the shadow's R0 back and reports the acknowledgement.
- The async form of `AudioCodec`.

## Where the traits fall short

- `AudioFraming` has two framings; the part also has DSP mode A and B and a
  right-justified format. `set_dsp_format` covers DSP.
- `AudioCodec` has one output gain and one input gain. The part has a
  headphone volume, a speaker volume and two digital volumes; the extra
  ones are the driver's own functions.
- `AudioCodec` has no mute for the input path; the part has an input PGA
  mute, which `codec_set_power(true)` clears.
- `I2sConfig` has one `bit_depth`, which is both the slot width and the
  sample width.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `AudioCodec` trait this driver implements and the wiring record and
  register helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` and `I2s` traits and the board handles.
- [es8311-nv](https://novo-lang.org/packages/es8311-nv) drives a mono codec
  with the same trait.
- [inmp441-nv](https://novo-lang.org/packages/inmp441-nv) reads an I2S
  microphone over the same handle.

## Tests

`novo test` runs the module over a double of the part, a write-only register
file on an I2C bus whose log keeps every write and every pause, and a double
of the I2S handle: the control word, the shadow against the part's registers
after every sequence, the update bit that is never kept, the clock plan at
every rate and at Table 45 of the data sheet, the PLL for 12 MHz and 24 MHz
to 44.1 and 48 kHz, every step as an exact register log (the reset, the bias
up and down with its 100 ms and 600 ms waits, the outputs, the inputs, the
format with the PLL and its 250 ms lock, as master, and from a direct MCLK),
the volume pairs and their update bits, and the type over absent hardware.
Every refusal of a write at each position of the format, the power-up and the
power-down is checked. The suite covers every line of `src/`.

**Hardware: NOT VERIFIED.** See the example section.

The sources read for this driver, for their facts and not ported, are the
data sheet above, the Linux kernel's `sound/soc/codecs/wm8960.c` and NXP's
`drivers/audio/wm8960.c` in Zephyr. Embassy and esp-hal have no driver for
the part. The faults and differences, each named in `src/wm8960.nv` where it
is kept:

- Linux's capless bias path writes the anti-pop bits it means to clear with
  the mask and the value both set, so they stay on. Here the bits go to
  BUFIOEN alone.
- Zephyr's power-up writes VMIDSEL = 11, the 2 x 5k fast-start divider, with
  VREF and never moves to the playback divider, 2 x 50k (01). Its names for
  the divider values are not the data sheet's.
- Zephyr's "clear anti-pop" sets BUFDCOPEN, SOFT_ST, BUFIOEN, DISOP and DRES.
- Linux waits 250 ms after enabling the PLL, Zephyr 10 ms; the data sheet
  gives none, and the longer wait is taken.
- The data sheet's Table 45 and its worked example differ by one in K; the
  formula is taken.

## Licence

Apache-2.0.
