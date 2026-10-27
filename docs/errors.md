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
| `HOST_NOT_READY` | The host can't take a sale: the phone isn't enrolled, or it holds no host key that verifies. The card was not read. | Payments are being set up. Try again shortly. (Call `prepareHost`, and show amount entry only once it reports ready.) |
| `UNTRUSTED_HOST` | The host's key isn't certified by a root built into the SDK, has expired, or matches none of your pins. Nothing was sent. | Contact support. |
| `SYNTHETIC_PROFILE` | The built-in test profile was used with a real host. Nothing was sent. | (An app bug: load your own profile.) |

## `HostFailureReason`

`AcceptState.HostFailure` means the sale did not complete with your host. Its `transactionId` and `stan` name the sale when a request was sent.

| Reason | Meaning | Charged? | Suggested message |
|---|---|---|---|
| `NOT_CONFIGURED` | No host, key or activation is set up. | No | Payments are not set up on this device. |
| `ENROLMENT` | The phone could not enrol with the host (for example it needs a new activation code). | No | Contact support. |
| `TRANSPORT`, `TIMEOUT` | The host could not be reached before the request was sent, or it refused it with an HTTP error. | No | Try again. |
| `REJECTED` | The host refused the request at a named security check (in `detail`). | No | Contact support. |
| `UNDECIDED` | Not reported for a sent sale any more: an answer that cannot be checked is resolved by a status query instead. | No | Try again. |
| `REVERSED` | The answer was lost, the outcome stayed unknown, and the SDK reversed the sale; the host confirmed it. | No | Not charged. Please pay again. |
| `UNRESOLVED` | The answer was lost and the reversal is not confirmed yet. The SDK keeps sending it at every start and reports the end through `transactionListener`. | **Maybe** | The payment could not be confirmed. Do not pay again yet. |

A lost answer is resolved before you are told: if the host approved the sale, you get `Complete`, not a failure.

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
