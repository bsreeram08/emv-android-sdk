---
title: Configuration
nav_order: 4
---

# Configuration: one JSON document

The whole SDK is configured from one JSON document: the terminal profile and the tap feedback. Your app can ship the document, or fetch it from your own config service or remote-config tool. You can swap it at any time without restarting anything.

```kotlin
EmvSdk.load(json)                 // validate and apply; also warms the kernel
EmvSdk.prepareSale(amountMinor)   // before each sale
```

```json
{
  "version": 1,
  "profile": {
    "currencyCode": 752,
    "countryCode": 752,
    "transactionType": 0,
    "ttq": "20800000",
    "floorLimitMinor": 0,
    "cvmLimitMinor": 5000,
    "transactionLimitMinor": 10000,
    "terminalType": 22,
    "terminalCapabilities": "0008E8",
    "additionalTerminalCapabilities": "6000F0A001",
    "terminalId": "TERM0001",
    "merchantId": "MERCHANT0000001",
    "merchantCategoryCode": 5999,
    "merchantName": "EXAMPLE STORE",
    "aids": [
      {"aid": "A0000000031010", "scheme": "visa"},
      {"aid": "A0000000041010", "scheme": "mastercard"}
    ]
  },
  "feedback": {"sound": true, "vibration": true, "volume": 1.0, "outcomes": true}
}
```

| Field | Required | Meaning |
|---|---|---|
| `version` | yes | Must be `1`. |
| `profile` | yes | The terminal profile. See [Terminal profile](profile.md). |
| `host` | yes for live sales | `{"url": "https://…"}`, optionally with `"keySha256": "<fingerprint>"` or `["<current>", "<next>"]` as an extra pin. Release builds require `https`. See [Host](#host). |
| `feedback` | no | The tap sound and vibration (below). If the document has no `feedback`, the current settings stay as they are, so a cardholder's settings switch survives a configuration swap. |

## How loading behaves

- **All or nothing.** `load` checks the whole document before it changes anything, and the kernel checks the profile too. If either rejects it, the previous configuration stays in force.
- **Loading never touches a sale.** `load` only checks and stores the configuration. `prepareSale` installs it for one sale, so a configuration that arrives mid-sale can't change that sale's amount.
- **Live swap.** A new configuration applies from the next tap. A tap that is already in progress finishes with the configuration it started with.
- **Fast first tap.** The first `load` also starts the native kernel, so the customer's first tap doesn't wait for it. The tones are prepared when the payment screen opens, and cues play off the card path.
- **Errors you can act on.** A malformed document throws `EmvConfigException`. Its `field` names the problem, for example `feedback.volume`. The JSON must be strict: comments, single quotes, unquoted keys, duplicate keys at any depth, unknown fields, wrong types and data after the document are all rejected. The profile reaches the kernel exactly as you wrote it. When the kernel rejects the profile itself, `load` throws `SafeApduException`, with the EMV outcome when there is one.
- **Any thread.** Call `load` from any thread, for example right after your remote config arrives.

`prepareSale(amountMinor)` installs the profile with the sale amount (9F02, DE 4). Until the first `prepareSale`, sales use built-in placeholder terminal data, still with a fresh unpredictable number. If the profile doesn't set `utcOffsetMinutes`, it uses the device's time zone. Call it before `runAccept`.

## Host

```json
"host": {"url": "https://gateway.example.com"}
"host": {"url": "https://gateway.example.com", "keySha256": ["<current key SHA-256>", "<next key SHA-256>"]}
```

**Trust root.** Your gateway's key must carry a certificate issued from the vendor root that is built into the SDK. The SDK's payment core checks that certificate itself and seals card data to no other key, so a changed configuration or a network attacker cannot redirect card data. Your gateway serves the certificate; it renews and rotates its key without any change on the phone: the SDK fetches the new certified key by itself, and never enrols again for it.

**Pins (optional).** `keySha256` lists up to two SHA-256 fingerprints of the gateway key. With pins, a certified key must also match one of them. Two pins let you rotate with no outage; leave them out to rely on the trust root alone.

**Activation.** Each terminal is activated once with a one-time code your gateway issues for one terminal of one merchant (`XXXX-XXXX-XXXX-XXXX`). Pass it in code, never in the JSON:

```kotlin
EmvSdk.activate(code, merchantId = "M-100") // merchantId is optional: a code for another merchant is refused
```

The next sale that needs it enrols with the code. The gateway binds the terminal to the code's merchant and terminal ID; those, not the IDs in your terminal profile, are what your processor sees. The code is held in memory only, never logged or stored, and is forgotten once used. If the network drops the activation's answer, the next sale resends it and gets the same terminal. Codes are single use and expire (by default after 72 hours). A terminal the gateway no longer knows reports `ENROLMENT` and needs a new code; release builds never enrol again by themselves.

## Tap feedback

The SDK plays the two audio indications from EMV Contactless Book A, each with a matching vibration:

| Cue | When | Sound | Vibration |
|---|---|---|---|
| `TapCue.SUCCESS` | The card read has finished and the card can be removed. It plays before your host is called. | About 1500 Hz for 500 ms | One short pulse |
| `TapCue.ALERT` | The card read failed, for example the card was removed early or its data failed the checks. | Two 750 Hz beeps: 200 ms on, 200 ms off, 200 ms on | Two pulses |
| `TapCue.APPROVED` | Your host approved. Book A defines no tone for this, so it is vibration only. | None | Two short pulses |
| `TapCue.DECLINED` | Your host declined. | The alert tone | Two pulses |

No cue plays when your app cancels the run or the screen pauses. If the card read fails, the reader stays on, so the customer can simply tap again. When a sale finishes, the card connection ends, so the next sale always starts with a new tap. The phone's own NFC sound is turned off, because it plays when the card is first detected. That is too early, and it encourages customers to pull the card away.

| `feedback` field | Default | Meaning |
|---|---|---|
| `sound` | `true` | Play the tones. They follow the phone's system sound volume. |
| `vibration` | `true` | Vibrate with each cue. |
| `volume` | `1.0` | Scales the tone, from `0.0` to `1.0`. |
| `outcomes` | `true` | Also cue the host's answer (`APPROVED`, `DECLINED`). |

You can also change the feedback in code, for example from a settings switch. The next cue uses the new value:

```kotlin
EmvSdk.feedback = EmvSdk.feedback.copy(sound = false)
controller.playCue(TapCue.ALERT)   // play a cue for your own event, such as a decline
```

The SDK's manifest adds the `VIBRATE` permission to your app.
