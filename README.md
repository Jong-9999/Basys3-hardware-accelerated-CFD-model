  # D2Q9 lattice-Boltzmann CFD on the Basys 3

A 64 x 48 D2Q9 fluid simulation that runs entirely on the FPGA and draws itself on the VGA port.
A web page on the PC changes the flow speed, viscosity, view mode and the **shape of the object**
over the USB-UART, so nothing needs a new bitstream.

```
  PC browser (sw/lbm_control.html)
        | Web Serial, 115200 baud, binary packets
        v
  uart_rx -> uart_cmd -> parameters / geometry RAM / commands
                              |
                         lbm_core  (9 x 2 ping-pong block RAMs, 24-stage pipeline, 1 cell per clock,
                              |     blow-up guard)
                              | rho, jx, jy per cell
                         viz_engine -> display RAM -> display_path -> Vga_Display (your library) --+
                                                                    -> vga_sync_gen (alternative) -+-> VGA
                                                                                          sw[15] picks one
```

## Files

| Path | What |
|---|---|
| `rtl/lbm_top.v` | **Top module.** Ports match your `Basys3_Master.xdc` names. |
| `rtl/lbm_core.v` | The LBM engine (streaming, collision, boundaries, pipeline, blow-up guard). |
| `rtl/fmem.v`, `ram_sdp.v`, `mask_ram.v`, `sr_delay.v` | Memories and delay lines. |
| `rtl/viz_engine.v`, `display_path.v` | Speed / density / vorticity -> colour -> VGA pixels. |
| `rtl/uart_rx.v`, `uart_tx.v`, `uart_cmd.v` | UART and the command parser. |
| `rtl/btn_pulse.v` | Button debounce. |
| `rtl/vga_modules.v` | Your `vga-module` library, **unmodified** (`Vga_Display` / `vga_controller` are used). |
| `rtl/vga_sync_gen.v` | Alternative, conventional VGA timing generator, selected with `sw[15]` (see troubleshooting). |
| `rtl/vga_test_top.v` | **Test top, not part of the normal build.** VGA test pattern only, for isolating display problems. |
| `constraints/Basys3_Master.xdc` | Your constraints with `sw[15:0]`, `led[15:0]`, `btnC`, `btnR`, `RsRx`, `RsTx` uncommented (clock and VGA were already active). |
| `sw/lbm_control.html` | The control page (no install, no dependencies). |
| `sw/lbm_model.py` | Integer reference model of the core's arithmetic, including register-width tracking and the guard rules. |
| `sw/test_core.js` | Node tests for the page's protocol / geometry code: `node sw/test_core.js`. |
| `docs/yosys_resource_estimate.txt` | Raw synthesis statistics. |
| `ref/` | Throwaway clone of the vga-module repo that I used for reading. Not part of the build; safe to delete. |

## Vivado setup

1. New RTL project, part **xc7a35tcpg236-1** (Basys 3).
2. Add Sources: every file in `rtl/` (all plain Verilog) and `constraints/Basys3_Master.xdc` as the constraints.
3. **Set `lbm_top` as the top module.** `vga_modules.v` also contains `rectangle`, `circle`, `triangle` and `character`, and `vga_test_top.v` is a second top, so Vivado may otherwise pick the wrong one.
4. Synthesis, implementation, generate bitstream, program the board.

With no PC attached the board already shows a flow past a cylinder (the geometry in `mask_ram.v` and the defaults in `uart_cmd.v`).

## The control page

Open `sw/lbm_control.html` in **desktop Chrome or Edge** (Web Serial is not in Firefox or Safari). If double-clicking the file does not offer the serial API, serve it instead: `cd sw`, `python -m http.server`, open `http://localhost:8000/lbm_control.html`.

1. Click **Connect to FPGA** and choose the Basys 3's COM port. The page pings the board and then sends all settings and the geometry.
2. **Geometry** (left): paint with the brush, or switch to **Shape** to stamp circles, rectangles, wedges, diamonds and NACA airfoils with size and rotation. Right-click cuts a hole. Materials: Wall, Erase, Inlet, Outlet. Presets: cylinder, square, flat plate, wedge, airfoil, two cylinders, backward step, lid-driven cavity, empty channel. **Import image** turns dark pixels into walls. Ctrl+Z undoes. Changes are sent live, and only the bytes that changed are sent.
3. **Flow**: inlet speed and angle, relaxation rate omega (viscosity). The page shows nu, the object height, the Reynolds number, and warns when the speed is too high for the chosen omega.
4. **Simulation**: steps per video frame (0 = pause), re-initialise, single step, step counter, and the blow-up guard (below).
5. **Display**: Speed (dark blue = still, red = fast), Density (red = high pressure, blue = low), Vorticity (red = clockwise on screen). The colour-range slider sets the full-scale value; the legend shows the exact FPGA palette.

Board controls: `sw[0]` pause, `sw[1]` take the view from `sw[3:2]` (0 speed, 1 density, 2 vorticity), `sw[15]` VGA timing source (0 library, 1 built-in), `btnC` re-initialise, `btnR` single step.
LEDs: `[0]` running, `[1]` UART rx, `[2]` command ok, `[3]` command error, `[5:4]` view mode, `[6]` blow-up detected (flashes), `[7]` halted after repeated blow-ups, `[15:8]` step counter bits 15..8.

### Cell types

| Value | Type | Behaviour |
|---|---|---|
| 0 | Fluid | stream + BGK collide |
| 1 | Wall | full-way bounce-back (no-slip) |
| 2 | Inlet | forced to the inlet velocity at density 1. Use it for an inflow column, or as a moving lid |
| 3 | Outlet | **pressure outlet**: density held at the reference (1), velocity taken from the cell to its **west**. Keep it on the right edge |

The grid edges wrap around, so draw walls along any edge you want closed (the presets do). The inlet velocity is one global value, so a lid-driven cavity needs the angle at 0.

## When the screen goes solid red

That is a numerical blow-up: the populations or velocities left the range the fixed-point format can hold, they saturate, and a saturated field never recovers on its own. There were two causes, and both are handled.

2. **Settings the scheme cannot handle.** The lattice-Boltzmann method goes unstable at high speed combined with a high omega (thin fluid). The page warns about this and the table in "Numerics" gives the tested limits.

**Blow-up guard** (on by default, switch on the page): the core watches for any population that needs clamping or any velocity or density deviation reaching +-0.5. On a hit it rebuilds the flow field automatically (LED 6 flashes, the page counts the resets). After three failures in a row it stops on the clean initial field and the page shows a banner (LED 7); moving any control, painting, or pressing re-initialise restarts the run, and the field is rebuilt for the new settings. In the reference model, healthy runs never trip the guard, a hopeless setting (speed 0.25, omega 1.95) is caught on the third step, and a run resumes cleanly once the settings are safe.

I also checked for integer wrap-around specifically: the reference model tracks every pipeline register at its exact RTL width. No register ever overflowed, even in a blown-up run (the busiest register uses about 5 % of its range), and a model that wraps at those widths gives bit-identical results to an unbounded one.

## When the picture flashes or goes black

Nothing in the simulation drives the sync signals, so this is most likely a VGA-timing or monitor issue rather than the simulation. The library's `vga_controller` uses a LUT output as a clock for its counters and produces positive-polarity syncs; that is fragile in a large design and some monitors dislike it. I cannot see your hardware, so here is how to find out:

1. **Flip `sw[15]` up** on the running design. This swaps in `vga_sync_gen.v`: one clock domain, standard 640x480@60 timing, negative-polarity syncs. If the flashing stops, it was the library's timing or polarity.
2. **Rule out the simulator:** set `vga_test_top` as the top module (same constraints file), build, and look at the colour bars with `sw[15]` down and up. The white box moves one pixel per frame, so a glitch is easy to see. If it flashes here as well, the simulation is not involved.
3. **If both timing sources flash, even in the test pattern:** try another cable, another VGA port or adapter, or another monitor.
4. **Check Vivado's timing report** (`report_timing_summary`): any negative worst slack, especially hold slack on paths into the `r_25MHz` / `h_count` registers of the library, would explain random glitches.

## UART protocol

8N1, 115200 baud by default. Every packet is `0xA5`, command, arguments. Multi-byte values are big-endian. Bytes before `0xA5` are ignored and a half-received packet is dropped after about 84 ms, so the link always recovers.

| Cmd | Name | Arguments | Reply |
|---|---|---|---|
| `0x01` | PING | none | `4C 42 4D 01` ("LBM", version) |
| `0x02` | SET_REG | reg, hi, lo | none |
| `0x03` | CMD | op: 0 re-initialise, 1 single step | none |
| `0x04` | WRITE_MASK | addr_hi, addr_lo, len_hi, len_lo, data... | none |
| `0x07` | GET_REG | reg | hi, lo |

Geometry: cell `p = y*64 + x` is in byte `p>>2`, bits `[2*(p%4)+1 : 2*(p%4)]` (768 bytes in total).

| Reg | Meaning | Format |
|---|---|---|
| `0x00` | OMEGA_E | Q2.14, `1.0 = 16384`; nu = (1/omega - 0.5)/3 |
| `0x01` | OMEGA_O | Q2.14, equal to OMEGA_E for plain BGK |
| `0x02` / `0x03` | UIN_X / UIN_Y | signed Q0.16, `0.1 = 6554` |
| `0x04` | SPF | steps per video frame, 0 = paused |
| `0x05` | VIZ_MODE | 0 speed, 1 density, 2 vorticity |
| `0x06` / `0x07` / `0x08` | GAIN0 / 1 / 2 | colour scale per mode |
| `0x09` | GUARD | 1 = blow-up guard on (default) |
| `0x10` / `0x11` | STEP_LO / STEP_HI | read only |
| `0x12` | STATUS | read only: bit 0 sweeping, bit 1 running, bit 2 halted after repeated blow-ups |
| `0x13` | DIV_COUNT | read only: automatic re-initialisations so far |

Any SET_REG, CMD or WRITE_MASK also releases the halted state. After PING or GET_REG wait for the reply before sending the next packet (the FPGA receiver has no FIFO).

To change the baud rate, change `CLKS_PER_BIT` in `lbm_top.v` (clock / baud, e.g. 109 for 921600) and pick the same rate in the page.

## Resources on the Basys 3 (XC7A35T)

From a Yosys `synth_xilinx` run of the whole design (including the guard and both VGA timing sources). Vivado's mapping will differ a little; treat these as estimates.

| Resource | Used | Available | |
|---|---|---|---|
| LUTs (logic) | about 3,100 | 20,800 | 15 % |
| LUTs used as shift registers (SRL, counted in addition) | about 550 | 6,400 | 9 % |
| LUTs in total | about 3,600 | 20,800 | 17 % |
| Flip-flops | about 3,000 | 41,600 | 7 % |
| **Block RAM** | **33 RAMB36 equivalents (64 x RAMB18 + 1 x RAMB36)** | **50** | **66 %** |
| DSP48E1 | 25 | 90 | 28 % |
| I/O | 51 | 106 | |

Block RAM is the limiting resource. The 18 distribution banks (9 directions x 2 ping-pong) take 54 RAMB18; the macroscopic buffers, display buffer and geometry take the rest. 64 x 48 is therefore close to the largest grid this architecture fits; the logic and DSPs have plenty of room.

## Performance

One cell per clock: a step is 3,072 cells plus about 30 cycles of pipeline, about 31,000 steps per second, up to about 530 per 60 Hz frame. The default of 16 steps per frame is about 1,000 steps per second, so a vortex street takes a few seconds to appear. Raise the slider to run faster.

## Numerics

* Incompressible LBM (He-Luo): rho0 = 1, u = j, so there is no division.
* Populations stored as deviation from rest, signed Q2.16 in the 18-bit block RAM words. The rest population is set from mass conservation, so rounding cannot create or destroy mass (in a closed box the reference model's total mass stays exactly constant).
* BGK collision, written as even/odd parts so a separate odd rate exists in a register. TRT with a small odd rate (below about 0.2) was unstable in fixed point, so the page does not offer it.
* Tested stability (reference model, 64 x 48 cylinder channel, 8,000 steps, inlet speed against omega):

  | omega \ inlet speed | 0.05 | 0.10 | 0.15 | 0.20 | 0.25 |
  |---|---|---|---|---|---|
  | 1.50 | 0.09 | 0.16 | 0.24 | 0.32 | 0.39 |
  | 1.70 | 0.09 | 0.17 | 0.24 | 0.33 | **unstable** |
  | 1.85 | 0.09 | 0.18 | 0.28 | **unstable** | **unstable** |
  | 1.90 | 0.09 | 0.19 | 0.32 | **unstable** | **unstable** |
  | 1.95 | 0.10 | 0.19 | **unstable** | **unstable** | **unstable** |

  (Entries are the peak local speed reached; the lattice limit is about 0.3 to 0.4 for well-behaved flow.) The page warns above 0.25 for omega up to 1.55, 0.20 up to 1.75, 0.15 up to 1.92 and 0.10 above that. The defaults (speed 0.10, omega 1.88) sit 1.5x below the limit. The limits depend on the geometry (a thicker obstacle speeds the flow up), so treat them as a guide; the blow-up guard is the safety net.
