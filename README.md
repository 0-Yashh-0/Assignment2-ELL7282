# ELL7282 / ELL782 – Five-Stage Washing-Machine Microcontroller

## 1. Project Overview

This project implements an 8-bit, 16-bit-instruction, five-stage pipelined processor in Logisim-Evolution for a washing-machine controller. The pipeline stages are IF, ID, EX, MEM, and WB. The processor uses eight 8-bit general-purpose registers (R0-R7), a 256 x 16 instruction ROM, and a 128-byte data RAM.

The design includes forwarding, a one-cycle load-use stall, branch/jump flushing, `HALT` handling, memory-mapped I/O, an independent timer, and a motor safety interlock.

## 2. Logisim-Evolution Version

Use Logisim-Evolution. The exact installed version was not provided in the project materials. Before submission, open Help > About and record the installed version here:

Logisim-Evolution version: "logisim-evolution-5.0.0"

## 3. Instruction Rom Setup

The supplied `program.hex` is a Logisim v2.0 raw memory image.

1. Open the final .circ project in Logisim-Evolution.
2. Select the circuit containing the processor and external washing-machine pins. In the current design this is the integrated `PIPELINE_TOP` circuit (or the outer wrapper, if one is used).
3. Open `IF_STAGE` and select the instruction ROM.
4. Configure the ROM for 8 address bits and 16 data bits (256 x 16).
5. Open the ROM contents editor and use the Load Image command to load `program.hex`.
6. Confirm that instruction words are loaded from address 00 through address 6A. The last instruction is `HALT` (`C000`) at address 6A.
7. Save the circuit file.

## 4. Clock And Reset Setup

Use one global rising-edge clock for the processor, pipeline registers, register file, MEM-stage peripheral latches, timer, and `MOTOR_SAFETY` circuit.

`RESET` is active high:
- Assert `RESET`=1 and apply a rising clock edge to reset PC and invalidate pipeline entries. Actuator latches, timer state, and `TRIP` must also reset.
- Release `RESET`=0 before normal execution.
- Start the global clock only after reset has been released.

Use exactly one clock driver on the global `CLK` net:
- For interactive simulation, use the Logisim Clock component as the clock source.
- For Test Vector simulation, drive `CLK` from the top-level test-vector input. Disable or disconnect the separate Clock component from that same net.
- Never connect a Clock component and a test-vector `CLK` output to the same wire at the same time; this can create E (error) values.

Likewise, `RESET` should have one driver. Do not connect a constant and an input pin or two output sources to the same `RESET` net.

## 5. External Inputs

The washing-machine interface uses:

- `START` (1 bit): software start request. Keep it asserted until accepted by the program.
- `MODE` (2 bits): selected mode. `MODE` is sampled when `START` is accepted; later changes should not change the current cycle's parameters.
- `DOOR_CLOSED` (1 bit): 1 means closed; use 1 for normal end-to-end wash tests.
- `WATER_LEVEL` (8 bits): unsigned water-level sensor input.
- `TEMP` (8 bits): temperature readback input; keep stable while sampled. 
- `DRUM_RPM` (8 bits): RPM readback input; keep stable while sampled. 

Manually change `WATER_LEVEL` while the program runs.

## 6. Running The Wash Program Interactively

Before starting:
- Load `program.hex` into the instruction ROM.
- Assert `RESET`=1, set `START`=0, set `DOOR_CLOSED`=1, and select the desired `MODE`.
- Set initial sensor values.
- Apply a reset clock edge and then release `RESET`=0.
- Start the global clock. Use manual full-cycle ticking initially so that PC, pipeline valid bits, timer, and outputs can be inspected.

`MODE` 00 test:
- Set `MODE`=00 and `DOOR_CLOSED`=1.
- Hold `START`=0 during reset, then set `START`=1 and keep it high until accepted.
- The program's fill target is 100 (64 hex).
- When `VALVE_IN` becomes 1, hold `WATER_LEVEL`=99 initially. Then change it to 102. The valve should turn off when the level is at least 100; the sensor need not report exactly 100.
- During drain, when `PUMP_OUT` becomes 1, set `WATER_LEVEL` to 12, then change it to 10. The pump should stop at or below 10.
- Observe wash direction changes, timer count/status, spin interval, and final outputs.

`MODE` 01 test:
- Reset the processor before the next run.
- Set `MODE`=01, `DOOR_CLOSED`=1, and start the program.
- The fill target is 150 (96 hex).
- When `VALVE_IN` becomes 1, use `WATER_LEVEL`=149, then change it to 151. The valve should turn off at or above 150.
- During drain, use 12 then 10 as in `MODE` 00.
- Observe the longer `MODE` 01 wash/spin timer intervals.

Both modes use a drain threshold of 10 (0A hex). Heater remains off in the core demonstration.

## 7. Expected Completion Outputs

After a successful run, the intended final values are:
- `VALVE_IN` = 0
- `HEATER` = 0
- `CPU_MOTOR_EN_REQ` = 0
- `MOTOR_EN` (safety-gated actual output) = 0
- `MOTOR_DIR` = 0
- `PUMP_OUT` = 0
- `DOOR_LOCK` = 0
- `DISPLAY` = 01
- `BUZZER` = 1
- `TRIP` = 0, assuming no safety event occurred
- CPU reaches the valid `HALT` instruction at ROM address 6A

The timer MATCH event occurs on its programmed count. The processor polls `TIMER_STATUS` and then executes the motor-off store; therefore the motor request latch is cleared after a short software/pipeline response latency, not on the exact timer-match edge.

## 8. Test-Vector Procedure

To run a component or top-level test vector:
1. Open the circuit being tested (for example `CONTROL_UNIT`, `PERIPH_ADDR_DECODER`, `MEM_STAGE`, `MOTOR_SAFETY`, or `PIPELINE_TOP`).
2. Open Simulate -> Test Vector.
3. Load the matching .txt vector file.
4. Ensure that the vector file declares every input and output pin with the exact circuit pin name and bit width.
5. Use Show to display the selected sequence row.
6. For top-level vectors, let the vector file drive `CLK`. Disable the separate Clock component from the same clock net.