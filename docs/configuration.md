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
| `host` | yes for live sales | `{"url": "https://…", "keySha256": "<fingerprint>"}` or two fingerprints `["<current>", "<next>"]` for key rotation. Release builds require `https`. See [Host](#host). |
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
"host": {"url": "https://gateway.example.com", "keySha256": ["<current key SHA-256>", "<next key SHA-256>"]}
```

The SDK only enrols with, and only seals card data to, a host whose key matches one of the pinned fingerprints. Two pins let your gateway rotate its key with no outage. The enrolment token for the first enrolment is set in code (`EmvSdk.enrolmentToken`), never in the JSON.

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
