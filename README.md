# VoodooI2CHID (macOS 26 fork)

Fork of [VoodooI2CHID](https://github.com/VoodooI2C/VoodooI2CHID) — the I2C-HID satellite for
[VoodooI2C](https://github.com/Movesxml/VoodooI2C). Used to bring up an **ELAN0001 precision
touchpad** on a Lenovo Legion Y7000P IRH8 under macOS 26.

## What changed

The upstream probe is fragile: a single failed HID descriptor read dooms the whole boot, and in
polling mode an interrupt event source can be registered that never fires.

| Change | Why |
|---|---|
| HID descriptor read retried (~1 s total) | Some devices need time after `_PS0` / reset before answering |
| Fallback to register `0x0001` when the HIDG `_DSM` is missing | HID-over-I2C always exposes the descriptor at `0x0001` |
| Explicit timer polling when VoodooI2C reports `Interrupt Mode = Polling` | An interrupt source that registers but never fires leaves the device silent |
| Host-initiated reset no longer blocks the work loop for 12 s in polling mode | The polling timer lives on that work loop and could not run |
| Report descriptor read retried (5 × 100 ms) | Same readiness problem as above |
| Allocation and return-value checks in report paths | Avoid null dereferences under memory pressure |
| Division-by-zero guards when parsing contact counts | Malformed descriptors could panic the kernel |
| PTP mode switch written twice + logged | Some firmwares ignore the first switch request |
| More diagnostics | HID descriptor contents, element counts, interrupt mode |

The changes are limited to robustness; the HID protocol implementation is unchanged.

## Installation

Install together with the core:

```
VoodooI2C.kext/Contents/PlugIns/VoodooI2CServices.kext
VoodooI2C.kext/Contents/PlugIns/VoodooGPIO.kext
VoodooI2C.kext/Contents/PlugIns/VoodooInput.kext
VoodooI2C.kext
VoodooI2CHID.kext
```

`VoodooI2CHID.kext` must be listed in `config.plist → Kernel → Add` — the core alone produces no
touchpad.

## Building

```bash
xcodebuild -project VoodooI2CHID.xcodeproj -scheme VoodooI2CHID \
           -configuration Release CODE_SIGNING_ALLOWED=NO
```

The project expects `MacKernelSDK` next to it (CI checks it out), same as upstream.

## Diagnostics

```bash
ioreg -c VoodooI2CHIDDevice -l | head -40
log show --last boot --debug --predicate 'eventMessage CONTAINS "VoodooI2CHID"' | tail -40
```

Look for:

```
HID descriptor: VID 0x04f3 PID … report descriptor … bytes
Multitouch elements: N finger collection(s) … input mode yes
Device switched to Precision Touchpad mode
```

## Credits

* [Alexandre Daoud / coolstar](https://github.com/VoodooI2C) — original VoodooI2CHID
* [kprinssu](https://github.com/kprinssu), [1Revenger1](https://github.com/1Revenger1) — upstream maintenance
* [acidanthera](https://github.com/acidanthera) — MacKernelSDK, VoodooInput

## License

GPL-3.0, inherited from VoodooI2C.
