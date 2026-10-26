---
title: Terminal profile
nav_order: 5
---

# Terminal profile

The profile is the `profile` object of the [configuration document](configuration.md). The SDK sets `amountMinor` for each sale with `EmvSdk.prepareSale`, so you can leave it out of the document. `AcceptProfile.configure(json)` also takes a profile on its own; it checks the profile and applies it to every sale started afterwards. An invalid profile throws `SafeApduException`. When the cause is an EMV outcome, the exception's `outcome` says which one, for example a profile that allows no card. The SDK rejects unknown fields and anything after the JSON document.

## Example

```json
{
  "amountMinor": 1000,
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
  "utcOffsetMinutes": 60,
  "aids": [
    {"aid": "A0000000031010", "scheme": "visa"},
    {"aid": "A0000000041010", "scheme": "mastercard"}
  ]
}
```

## Fields

| Field | Type | Meaning |
|---|---|---|
| `amountMinor` | integer | Sale amount in minor units (cents, öre). `EmvSdk.prepareSale` sets it for each sale. |
| `currencyCode` | integer | ISO 4217 numeric currency code |
| `countryCode` | integer | ISO 3166 numeric country code of the terminal |
| `transactionType` | integer | `0` for a purchase |
| `ttq` | hex, 4 bytes | Terminal Transaction Qualifiers. The SDK sets the online-required and CVM-required bits for each sale. |
| `floorLimitMinor`, `cvmLimitMinor`, `transactionLimitMinor` | integer | Contactless limits in minor units |
| `terminalType` | integer | EMV terminal type, for example `22` (attended, online only, merchant) |
| `terminalCapabilities` | hex, 3 bytes | Terminal Capabilities |
| `additionalTerminalCapabilities` | hex, 5 bytes | Additional Terminal Capabilities |
| `terminalId` | 8 characters | |
| `merchantId` | 15 characters | |
| `merchantCategoryCode` | integer | MCC |
| `merchantName` | string | |
| `utcOffsetMinutes` | integer | The device's time-zone offset, used for the transaction date and time |
| `date`, `time` | `YYMMDD`, `HHMMSS` | Optional fixed date and time, for tests only. Leave them out in live use. |
| `aids` | list | The applications the terminal accepts (below) |

The profile cannot set the unpredictable number. A profile that tries is rejected.

### `aids` entries

| Field | Meaning |
|---|---|
| `aid` | Application identifier in hex, 5 to 16 bytes |
| `scheme` | `"visa"` or `"mastercard"` |
| `kernel` | Kernel ID. Leave it out to use the scheme's default. |
| `ttq`, `floorLimitMinor`, `cvmLimitMinor`, `transactionLimitMinor` | Override the terminal-wide values for this AID |
| `extendedSelection`, `statusCheck`, `zeroAmountAllowed` | Entry Point options for this AID |
| `kernel8` | Kernel 8 configuration as a map of tag hex to value hex. Required for Kernel ID 8. |

A duplicate AID is rejected.
