# Free-space marker (demo patch)

A demo firmware patch that writes and reverts an inert marker in empty flash.

## What this plugin does

Writes 8 bytes (`52 4F 4D 52 45 56 56 52` — "ROMREVVR" in ASCII) into a region
of **empty flash** on the 60E1B900 calibration, and reverts them exactly on
uninstall. It is deliberately **inert**: nothing in the firmware jumps to those
bytes, so the ECU never executes them. It exists purely to demonstrate what a
firmware-patch plugin looks like, safely.

## Why install it

- A safe way to see the firmware-patch trust-preview and undo flow in action
  before trying a patch that changes real behaviour.

## Capabilities

**Patch-firmware.** Before install you'll see the exact address and bytes it
writes, and the original bytes it will restore on uninstall.
