# Tools

Small scripts for poking at the running DVR. All of them are read-only with
respect to flash — they talk to devices and `/proc`, and write nothing to SPI,
NAND or any yaffs2 filesystem.

| Script | What it does |
|---|---|
| [`led-scanner.exp`](led-scanner.exp) | Sweeps the front-panel LEDs, Knight Rider style |

## `led-scanner.exp`

```sh
./led-scanner.exp [mode] [cycles] [delay] [host]
```

| Argument | Default | Meaning |
|---|---|---|
| `mode` | `comet` | `comet` = head plus a one-LED trail; `single` = one LED at a time |
| `cycles` | `10` | Full left-right-left sweeps |
| `delay` | `0.28` | Seconds between steps |
| `host` | `dvr` | DVR hostname or address |

Examples:

```sh
./led-scanner.exp                  # comet, 10 sweeps
./led-scanner.exp single 20 0.15   # single LED, faster, twice as long
```

It exists as an executable example of the AT89S52 command 2 protocol in
[../doc/20-front-panel-mcu.md](../doc/20-front-panel-mcu.md#front-panel-leds).
Every frame is generated from the LED bit map with the checksum computed, not
read from a hardcoded table, so if the documented bit assignments or checksum
rule were wrong the sweep would visibly break.

### Reading the output

Two self-checks are printed.

`ENCCHECK` is preceded by a hex dump of the whole frame sequence. busybox
`printf` parses `\0240` as `\024` followed by a literal `0`, silently emitting a
malformed 6-byte frame, so the dump confirms the bytes before anything is sent.
Every frame must be exactly 5 bytes starting `a0 02`.

`TXDELTA` is the UART1 transmit counter from `/proc/tty/driver/ttyAMA` across
the run. It must equal `cycles x steps x 5`, plus 5 for the restore frame —
which is what `EXPECT` prints. Excess is the vendor application's own traffic,
normally a 30-second watchdog kick sent two or three times by the retry race
described in
[the protocol document](../doc/20-front-panel-mcu.md#why-every-command-appears-twice).
A shortfall means frames were malformed.

### Caveats

The vendor application owns `/dev/ttyAMA1` and reasserts its own LED state on
its next update, so the effect lasts only until something on the DVR changes a
status light. Running this puts a second writer on the bus; that is harmless,
since the commands are idempotent, but the vendor's ack matching will see
echoes of frames it did not send.

The panel is restored to `0x60` — `REC` + `HDD`, the normal idle state — on
exit.

`delay` is not the step period. busybox adds roughly 20 ms of loop overhead per
iteration on this box, so the default `0.28` lands at about 300 ms per step.
