---
title: Errors and outcomes
nav_order: 8
---

# Errors and outcomes

Every failure is a `SafeError`. Its `message` is short and safe to show, and it never contains platform exception text or card data. When the EMV kernel ended the transaction itself, `AcceptState.Failure.outcome` carries the EMV Outcome as well.

## `SafeError`

| Error | Meaning | Suggested message to the customer |
|---|---|---|
| `NFC_UNAVAILABLE` | The phone has no NFC adapter. | Card payments aren't available on this phone. |
| `NFC_DISABLED` | NFC is switched off. | Turn on NFC to pay by card. |
| `UNSUPPORTED_TAG` | The tag is not ISO-DEP, so it isn't a payment card. | Use a contactless payment card. |
| `CONNECT_TIMEOUT` | The card didn't connect within 2 s. | Hold the card still and try again. |
| `TRANSCEIVE_TIMEOUT` | The card didn't answer in time. | Try again. |
| `REMOVED` | The card left the field. | Hold the card still until the result shows. |
| `IO` | NFC communication failed. | Try again. |
| `CANCELLED` | Your app cancelled the run. | |
| `STALE_SESSION` | The call used a `sessionId` from an earlier tap. | (An app bug: use the latest `Ready` session.) |
| `BUSY` | A run is already in progress. | (An app bug: wait for the current result.) |
| `CLOSED` | The controller is closed. | (An app bug.) |
| `MALFORMED_RESPONSE` | The card sent a response that can't be parsed. | Try another card. |
| `PROTOCOL_CARD` | The card's data failed the kernel's checks. | Try another card. |
| `PROTOCOL_STATE` | The kernel reached an invalid state, or the profile is invalid. | Contact support. |
| `COMMAND_TOO_LONG`, `INVALID_COMMAND` | A command can't be sent to this card. | Contact support. |
| `ABI_MISMATCH` | The native library doesn't match this SDK build. | (A packaging error: reinstall.) |
| `INTERNAL` | An unexpected internal error. | Contact support. |

## `AcceptOutcome`

When the kernel ends a transaction, it gives an EMV Outcome:

| Field | Meaning |
|---|---|
| `status` | `END_APPLICATION`, `SELECT_NEXT`, `TRY_AGAIN`, `TRY_ANOTHER_INTERFACE`, `APPROVED`, `DECLINED`, or `ONLINE_REQUEST` |
| `start` | Where the transaction restarts, if anywhere (`A` to `D`) |
| `messageId` | The EMV user-interface message to show, for example `MESSAGE_INSERT_SWIPE_OR_TRY_ANOTHER_CARD` (`0x1C`) |
| `uiStatus` | The EMV reader status, for example `PROCESSING_ERROR` |
| `l1`, `l2` | The error source: `l1` for the radio link (time-out, transmission, protocol), `l2` for card data, status bytes, parsing, or terminal data |
| `sw` | The card's status word when `l2 = STATUS_BYTES`; otherwise `0` |

Show the message named by `messageId` when there is one. `TRY_ANOTHER_INTERFACE` means the profile or the card allows no contactless path for this sale.
