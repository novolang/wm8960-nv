# Changelog

## 0.1.0

The first release: a driver for Cirrus Logic's WM8960 stereo audio codec with
headphone and class-D speaker drivers, on an I2C control bus and a board's
I2S handle.

- `Wm8960`, the type `attach` builds from the board's I2C bus, the wiring
  and the I2S handle, implementing devices-nv's `AudioCodec`, with its own
  methods for the digital volumes, each output driver's volume, the
  microphone boost and bias, the speaker boost, DSP mode and register
  access.
- The control word: seven bits of register number and nine of data, as two
  bytes, and a shadow of the 56 write-only registers that starts from the
  data sheet's reset values, takes every write, never keeps a volume
  register's update bit, and is what every field update and every read-back
  uses. The shadow is module state, so the module holds one part's state.
- The clock plan: SYSCLK from MCLK directly, MCLK halved or the PLL, with the
  data sheet's fractional formula (N and K) for MCLK of any frequency whose N
  is 6 to 12, the ADC and DAC dividers for the nine sample rates, the
  class-D divider that gives 768 kHz or 705.6 kHz, and the bit-clock divider
  for a master codec. Tests pin it for 12 MHz and 24 MHz to 44.1 and 48 kHz
  and check Table 45 of the data sheet to within one count of K.
- The audio interface in I2S and left-justified formats through
  `AudioCodec`, and DSP mode A and B through `set_dsp_format`; 16, 24 and 32
  bits. `codec_set_format` configures and enables the board's I2S handle
  first.
- The headphone and speaker volumes and the input PGAs in thousandths of a
  decibel, never above the request, written as the data sheet's Figure 16
  shows with the left update bit clear and the right set, with zero-cross
  updates as an option.
- The bias up and down in the order of the anti-pop registers' bits, the
  outputs and the inputs, as named steps (`step_reset`, `step_base`,
  `step_bias_up`, `step_bias_down`, `step_outputs_up`, `step_inputs_up`,
  `step_power_up`, `step_power_down`, `step_interface`, `step_clocks`). The
  DAC is soft-muted through a power-up and the class-D outputs wait for a
  valid clock.
- The suite runs over a double of the part and of the I2S handle and covers
  every line of `src/`.
- The device program `examples/playback.nv`. Built for the nRF52840-DK it is
  20,032 B of flash and for the nRF54LM20-DK 21,392 B; before the Nordic buses
  landed, with the reading path left out, it was 14,108 B and 15,436 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries a WM8960, and no
  board package has an I2S block. The Nordic DKs drive I2C with TWIM0 and
  TWIM22, so the device program's images carry the reading path; a reply from
  the codec over either bus is NOT VERIFIED. The device program is given a
  stand-in handle.
- Rev 4.4 of the data sheet holds no power-up or anti-pop sequence. The order
  is from the bits' descriptions and the Linux and Zephyr drivers; the waits
  (100 ms for VMID, 250 ms for the PLL, 600 ms for the headphone discharge,
  100 ms after the reset) are theirs. Zephyr and Linux differ on the PLL
  wait, and the longer is taken.
- Table 45 of the data sheet and its worked example differ by one in K for the
  12 MHz to 12.288 MHz row; the formula is taken.
- The manifest names devices-nv and embedded-hal-nv by the registry range
  `^0.3.3`, the first releases with `AudioCodec` and `I2s`.
- Not implemented: the ALC, limiter and noise gate, 3D enhancement,
  de-emphasis, companding, jack detection, thermal shutdown, the 20-bit word
  length, the right-justified format, OUT3 and the mono output,
  `Identifiable` (the part cannot be read), and `AudioCodecAsync`.
- A second WM8960 in one program would share the shadow.
- The linter's `char-literal` rule reads register numbers such as 0x30 as
  characters. It is wrong for a register map, and those sites are left.
