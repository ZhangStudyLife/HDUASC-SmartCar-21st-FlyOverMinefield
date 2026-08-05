# Core 0 Scheduling, AirComm, and ToF Design

## Scope

Keep the existing bare-metal superloop, AirComm protocol, UART baud rate and pins, and the four-channel software I2C ToF wiring. The change only addresses low-frequency scheduling concentration, blocking AirComm transmission, ToF read latency, and repeated use of stale ToF samples.

## Wall-clock scheduler

Use `tick_1000us_cnt` as a 1 ms wall clock and dispatch one of ten low-frequency slots. A missed slot is discarded instead of replayed after a stall.

| Slot | Work |
| --- | --- |
| 0, 2, 4, 6, 8 | Advance the ToF transaction state machine once |
| 1 | Consume the latest ToF snapshot, update CRSF and landing input, run `FC_Loop_100Hz()` |
| 3 | Run AirComm 100 Hz maintenance, publish attitude, poll image IPC |
| 5 | Run beacon loss detection and planners, enqueue AirComm run data |
| 7 | Maintain IPC flight state and alternate the existing 50 Hz tasks |
| 9 | Advance the existing divider and run the takeoff/landing state machine at 10 Hz |

The 1 kHz control loop remains highest priority. Low-frequency work runs only after the current 1 kHz backlog is drained.

## AirComm transmission

Use the SCB4 TX FIFO interrupt rather than PDMA. PDMA would require a descriptor chain because the protocol maximum frame is 259 bytes while one descriptor count is limited to 256 bytes.

AirComm owns four fixed 259-byte frame slots. `RUN_DATA` may occupy at most three slots so that a heartbeat or command acknowledgement can use the fourth. Queue-full handling rejects the new frame without blocking and without incrementing the protocol sequence. The TX ISR fills the FIFO in batches and disables the TX trigger when no frame remains. RX and TX handling coexist in the UART2 ISR.

## ToF state machine

Split one measurement into complete software I2C transactions:

```text
READ_READY -> READ_STATUS -> READ_DISTANCE -> CLEAR_AND_PUBLISH -> IDLE
```

Each scheduler call performs at most one transaction. Runtime GPIO reconfiguration is removed. Synchronous I2C helpers receive a bus mask so status, distance, and interrupt-clear operations affect only sensors that were ready in this measurement.

Read `GPIO_HV_MUX_CTRL` register `0x0030` during initialization and derive each sensor's active ready level from bit 4. Runtime ready detection compares only bit 0 of `GPIO_STATUS`.

## Freshness semantics

The published ToF snapshot contains `ready_mask`, `fresh_mask`, and `sample_seq` in addition to distance and validity arrays.

- `ready_mask` identifies sensors that reported a new measurement.
- `fresh_mask` identifies channels updated by the published snapshot, including newly invalid measurements.
- `sample_seq` increments only after the selected sensors have been read and their interrupts cleared successfully.
- A clear failure does not publish the partially completed snapshot.

Height estimation compares `sample_seq` with the last consumed sequence and only fuses valid channels in `fresh_mask`. When no sequence arrives, it advances as a no-measurement cycle instead of applying old distances again. Velocity correction uses the number of missed 100 Hz cycles as the elapsed measurement interval.

## Verification

- Static review confirms no blocking UART write remains in the AirComm frame path.
- Every low-frequency task has exactly one wall-clock phase and missed phases are not replayed.
- Every ToF step contains at most one complete software I2C transaction.
- A repeated identical distance still increments `sample_seq` and is consumed once.
- One missing or non-ready ToF does not invalidate fresh data from the other channels.
- AirComm queue overflow is observable and never blocks Core 0.

Per the project `AGENTS.md`, command-line IAR compilation is not run; the user performs the target build and reports the result.
