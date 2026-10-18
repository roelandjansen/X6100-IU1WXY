# libx6100

Clean C library for the Xiegu X6100 baseband (STM32) as seen from the Allwinner R16
running Linux. Based on stock APP 1.2.0 / BASE 1.1.9, independent of R1CBU code.

## Files

| File | Content |
|------|---------|
| `x6100.h` / `x6100.c` | library: I2C, register shadow + diff sync, calibration, UART flow |
| `x6100_regs.h` | register map and packing macros |
| `x6100_cal.h` | calibration block layout (0xF000, 816 bytes) |
| `x6100_info.c` | read-only check: BASE version, calibration, 3 s of flow telemetry |
| `x6100_rodump.c` | standalone raw dump of status and calibration blocks |
| `build.sh` | cross build on Tumbleweed with `zig cc` (static, musl, Cortex-A7) |

## Build

```
zypper install zig
bash build.sh
```

Output in `out/`: `libx6100.a`, `x6100_info`, `x6100_rodump`.

## Use

The library has no threads. Typical loop:

```c
x6100_i2c_open(&i2c, NULL);
x6100_ctl_init(&ctl, &i2c);
x6100_cal_read(&i2c, &cal);
x6100_cal_apply(&ctl, &cal, x6100_band_index(freq));
x6100_ctl_refresh(&ctl);

for (;;) {
	x6100_set_freq(&ctl, 0, freq);
	x6100_cal_apply(&ctl, &cal, x6100_band_index(freq));
	x6100_ctl_sync(&ctl);			/* writes only changed registers */
	x6100_flow_read(&flow, &frame);		/* 1 = new frame */
}
```

Calibration is applied by the host, not the STM32: call `x6100_cal_apply()` on every
band change.

## Safety

- Stop `x6100_ui_v100` before using the library; it drives the same bus every 1 ms.
- `x6100_info` and `x6100_rodump` never write registers.
- There is deliberately no function to write the calibration block.
- Test TX only into a dummy load.

## Not yet covered

- Filter registers 44–47 semantics beyond low/high Hz
- GPIO (backlight 143, CI-V 138, key 203, baseband reset 204, WiFi 357)
- Audio path (ALSA `hw:0,0`)
- Stock UDP remote protocol
