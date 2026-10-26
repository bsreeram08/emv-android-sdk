---
title: NFC diagnostics
nav_order: 7
---

# NFC diagnostics

These APIs help you build a support screen and tell apart the two cases behind "the card doesn't read": the phone never saw the card, or the card was seen but the transaction failed. Neither API carries card data: no tag ID, no card commands or answers, no PAN.

## Readiness

```kotlin
val status = NfcDiagnostics.status(context)
if (!status.canReadCards) showNfcHelp(status)
```

| Field | Meaning |
|---|---|
| `hasNfcHardware` | The phone declares NFC hardware. |
| `adapterPresent` | An NFC adapter is available to apps. |
| `enabled` | NFC is switched on in Settings. |
| `secureNfcEnabled` | Secure NFC (Android 10+): when it's on, NFC works only with the screen unlocked. `null` if the phone has no such setting. |
| `readerOptionEnabled` | The reader-mode switch some phones have (Android 15+). When it's `false`, the phone reads no cards. `null` if the phone has no such switch. |
| `batteryPercent` | Battery level. A nearly empty battery can weaken or stop the NFC field. |
| `powerSaveMode` | Battery saver is on. |
| `canReadCards` | Adapter present, NFC on, and the reader switch not off. |

## Tap events

Set a listener before `onResume`:

```kotlin
controller.diagnosticsListener = NfcEventListener { event ->
    when (event) {
        NfcEvent.ReaderArmed -> log("polling for a card")
        NfcEvent.ReaderDisarmed -> log("reader off")
        is NfcEvent.AdapterStateChanged -> log("NFC on = ${event.enabled}")
        is NfcEvent.TagDiscovered -> log("found ${event.technologies}, ISO-DEP=${event.isoDep}")
        is NfcEvent.Tap -> log(event.report.toJson())
        else -> Unit   // later SDK versions may add events
    }
}
```

| Event | When |
|---|---|
| `ReaderArmed` | Reader mode is on and the phone is polling. |
| `ReaderDisarmed` | Reader mode is off. |
| `AdapterStateChanged(enabled)` | NFC was switched on or off while your screen was open. |
| `TagDiscovered(technologies, isoDep, sak, atqa)` | The phone found a tag. `isoDep = false` means it can't be a payment card. `sak` and `atqa` (NFC-A only) identify the chip type, not the card. |
| `Tap(report)` | A card run's timing (below). |

Events arrive on the main thread. They are built only when a listener is set, and a tag event is sent only after the card connection has started, so diagnostics never slow down the card.

## Tap timing

`TapTimingReport` shows where one tap spent its time, from discovery to the last card answer. All times are in microseconds:

- `discoveryMicros`, `connectMicros`, `readyToRunMicros`, `queueMicros`, `nativeStartMicros`: the stages before the first card command.
- `apdus`: one entry per card command. Each entry has the instruction byte, the command and response lengths, the status word, the app's gap before the command, the radio time (`transceiveMicros`), and the kernel time (`nativeMicros`).
- `cardMicros`: the total card time.
- `appMicros`: the part the app and SDK add. It is everything except connecting and radio time.
- `end`: `online`, or a failure name.

`report.toJson()` gives one line of names and numbers, ready to attach to a support ticket. The host call is not timed.

## Troubleshooting

| What you see | Likely cause | What to try |
|---|---|---|
| `canReadCards = false` | NFC is off, the reader switch is off, or the phone has no NFC. | Open NFC settings (`Settings.ACTION_NFC_SETTINGS`). |
| `ReaderArmed`, then no `TagDiscovered` during a tap | The phone's NFC controller never detected the card. The SDK and kernel never ran. | Find the phone's NFC antenna and hold the card flat against it for 1–2 s. Charge the phone if the battery is low. Remove thick or metal cases. Try another card. |
| `TagDiscovered` with `isoDep = false` | The tag is not a payment card, for example a transit tag or a sticker. | Use a contactless payment card. |
| `TagDiscovered`, then `REMOVED` or `CONNECT_TIMEOUT` | The card moved away or the field was weak. | Hold the card still until the result shows. |
| `TagDiscovered`, then `AcceptState.Failure` with an `outcome` | The kernel ended the transaction. | See [Errors and outcomes](errors.md). |

When you report a problem, attach the `NfcStatus`, the event list, and the `Tap` JSON. See [Support](support.md).
