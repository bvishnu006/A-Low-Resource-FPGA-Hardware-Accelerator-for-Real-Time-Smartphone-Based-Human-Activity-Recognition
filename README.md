# FPGA Live Activity Classifier

Real-time activity classification on a Basys3 (Artix-7) FPGA. 

## Files

| File | Description |
|---|---|
| `top_live.sv` | Top-level module. Connects the UART receiver, frame assembler and classifier, and drives the LEDs and 7-segment display. |
| `uart_rx.sv` | 8N1 UART receiver (100 MHz clock, 115200 baud). Outputs one byte plus a one-cycle `valid` pulse. |
| `frame_assembler.sv` | Collects a `0xAA` start byte followed by 24 data bytes, unpacks them into 12 signed 16-bit features, and pulses `frame_done`. |
| `classifier.sv` | Linear classifier (6 classes x 12 features). Runs as a state machine: multiply-accumulate, bias add, then a sequential argmax. |
| `weights_mem_fixed.mem` | Weight ROM, 72 values (6 x 12, row-major) in hex, Q8.8 fixed point. |
| `bias_mem_fixed.mem` | Bias ROM, 6 values in hex, Q8.8 fixed point. |
| `tb_top_live.sv` | Simulation testbench. Sends two example frames over the `rx` pin and prints `class_out`. |

## Data flow

```
Host (Python) -> UART 115200 8N1 -> uart_rx -> frame_assembler -> classifier -> LEDs / 7-seg
```

Each frame is one window of data. A prediction is made automatically whenever a complete
frame is received.

## Frame protocol

- 25 bytes per frame: `0xAA` (start of frame) followed by 24 data bytes
- 24 data bytes = 12 x int16, little-endian, Q8.8 fixed point
- Feature order (from the testbench): mean, std, min, max for x, then y, then z

## Classes

| Index | Activity |
|---|---|
| 0 | WALKING |
| 1 | WALKING_UPSTAIRS |
| 2 | WALKING_DOWNSTAIRS |
| 3 | SITTING |
| 4 | STANDING |
| 5 | LAYING |

The FPGA outputs only the index (3-bit LED value and one 7-segment digit).

## Timing

- Frame transfer over UART: about 2.17 ms
- Classifier latency: about 31 clock cycles (about 310 ns at 100 MHz)
- Prediction rate is set by how often the host sends frames

## Simulation

1. Add all `.sv` files and both `.mem` files to a Vivado project.
2. Set `tb_top_live` as the simulation top.
3. Make sure the `.mem` files are in the simulation working directory so `$readmemh` can find them.
4. Run behavioral simulation.
