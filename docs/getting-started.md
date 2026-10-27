---
title: Getting started
nav_order: 3
---

# Getting started

## Requirements

| | |
|---|---|
| Android | 8.0 (API 26) or later |
| ABIs | `arm64-v8a`, plus `x86_64` for emulators. 32-bit ARM is not supported. |
| Hardware | An NFC reader that supports ISO-DEP (NFC-A or NFC-B) |
| Language | Kotlin or Java, with JDK 17 |

## Add the library

Licensed customers receive the AAR through signed delivery. Add it to your app module the way your delivery notes describe. The SDK has no other runtime dependencies.

Declare NFC in your app's manifest:

```xml
<uses-permission android:name="android.permission.NFC" />
<uses-feature android:name="android.hardware.nfc" android:required="true" />
```

If your app also runs on phones without NFC, set `android:required="false"`. Then check [`NfcDiagnostics.status`](diagnostics.md#readiness) before you offer card payments.

## Lifecycle

Use one controller for each payment screen. Card reading is on only while the screen is resumed.

```kotlin
class PayActivity : Activity() {
    private val codec by lazy { NativeApduCodec() }
    private val controller by lazy { NfcSessionController(codec, ::onSession) }

    override fun onCreate(savedInstanceState: Bundle?) {
        // Keep card data off screenshots and the recents screen.
        window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)
        super.onCreate(savedInstanceState)
        EmvSdk.load(configJson)   // see Configuration; any time before the first sale
        controller.diagnosticsListener = NfcEventListener { event -> /* optional, see Diagnostics */ }
    }

    override fun onResume() {
        super.onResume()
        controller.onResume(this)   // turns on reader mode
        showConnecting()
        controller.prepareHost { notReady ->   // enrols or refreshes the host key if needed
            if (notReady == null) showAmountEntry() else showHostNotReady(notReady)
        }
    }

    override fun onPause() {
        controller.onPause(this)    // turns off reader mode, ends any card run
        super.onPause()
    }

    override fun onDestroy() {
        controller.close()
        super.onDestroy()
    }

    private fun onSession(state: SessionState) {
        when (state) {
            SessionState.WaitingForTag -> showTapPrompt()
            is SessionState.Ready -> startSale(state.sessionId)   // a card is connected
            is SessionState.Failure -> showError(state.error)
            else -> Unit
        }
    }
}
```

Call `onResume` and `onPause` on the main thread. Every callback the SDK makes arrives on the main thread.

## Session states

`SessionState` covers the card connection, not the sale:

| State | Meaning |
|---|---|
| `Inactive` | Reader mode is off: the screen is paused, or the controller is closed. |
| `WaitingForTag` | Reader mode is on and the phone is polling for a card. |
| `Connecting(sessionId)` | The phone found a card and is connecting. |
| `Ready(sessionId, technology, maximumTransceiveLength)` | The card is connected. Start the sale with this `sessionId`. |
| `Exchanging(sessionId)` | A card run is in progress. |
| `Failure(sessionId, error)` | The connection ended. See [Errors](errors.md). |

Each card that touches the phone gets a new `sessionId`. A call with an old `sessionId` fails with `STALE_SESSION`. This means a late callback from an earlier tap can never act on the current card.

If NFC is switched off, the SDK reports `Failure(null, NFC_DISABLED)`. When NFC is switched back on while your screen is open, reader mode turns back on by itself.

Next: [Configuration](configuration.md), then [Accepting a payment](payments.md).
