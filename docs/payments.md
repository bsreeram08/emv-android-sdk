---
title: Accepting a payment
nav_order: 5
---

# Accepting a payment

## 1. Set the amount

Load your configuration once, as described in [Configuration](configuration.md). Then set the amount before each sale:

```kotlin
EmvSdk.prepareSale(amountMinor)
```

A profile you configure always draws a fresh unpredictable number for each transaction.

**The test profile never reaches a real host.** `AcceptProfile.configure("")` selects the built-in test profile. It pins the unpredictable number so tests are repeatable, which means a cryptogram made with it could be replayed. The SDK refuses to send a sale made with it to any host except the simulated one, with `SafeError.SYNTHETIC_PROFILE`. If you never load a configuration, sales use built-in placeholder terminal data with a fresh unpredictable number; load your own profile before going live.

## 2. Run the sale on the connected card

When `SessionState.Ready` arrives, call `runAccept`:

```kotlin
controller.runAccept(sessionId) { state ->
    when (state) {
        is AcceptState.Exchanging -> showProgress(state.stage)        // PPSE, SELECT, GPO, RECORDS, ...
        is AcceptState.OnlineRequest -> showCard(state.card)           // masked card; the success tone has played
        is AcceptState.Complete -> showResult(state.outcome, state.responseCode)
        is AcceptState.HostFailure -> showHostProblem(state.reason)
        is AcceptState.Failure -> showError(state.error, state.outcome)
    }
}
```

The card part usually takes a fraction of a second. Ask the customer to hold the card still until you receive `OnlineRequest` or a failure. Every state carries the `sessionId` of its sale, so one handler can serve every sale safely.

## 3. Your host

The SDK talks to your payment host itself. You configure where it is in the [configuration](configuration.md#host), and the SDK:

- enrols the phone once, with its hardware-attested device key;
- seals each request inside its payment core, signs it with the device key, and sends it;
- opens the host's sealed answer and reports it as `Complete` or `HostFailure`.

Your app never receives the card number, expiry, track data, cryptogram or the online request. It gets a `CardSummary` instead: scheme, application, masked card number (at most the first 6 and last 4 digits), amount and currency. The host side runs the gateway library we provide to licensed customers, which opens the sealed requests and connects to your processor.

- Once the request has been sent, the sale can no longer be cancelled; its answer always reaches its own sale.
- A transport problem ends as `HostFailure` (for example `TIMEOUT` or `TRANSPORT`) and never as an approval.
- Release builds only talk to an `https` host whose key matches the pinned fingerprints.

## 4. The result

`AcceptState.Complete` carries:

- `outcome`: `APPROVED` or `DECLINED`;
- `responseCode` and `body`: from your host;
- `recordsRead`;
- `dataRecordDigest`: a hash you can use as a receipt reference. It is not card data.

After any result, the card connection ends and the controller returns to `WaitingForTag`, so the next sale always starts with a new tap. If the card read fails (for example the card is removed too early), the reader stays on: ask the customer to tap again.

## Cancelling

```kotlin
val cancelled = controller.cancelActiveExchange(sessionId)
```

This ends only the card run in progress for that `sessionId`. It returns `false` when nothing is running, for example when the run has already finished. A cancelled run reports `Failure(CANCELLED)`, and nothing from it can complete a later sale.

## Threading and timeouts

- Each controller runs one card exchange at a time. A second `runAccept` while one is running fails with `BUSY`.
- Connecting to a card times out after 2 seconds. A single card command times out after about 5 seconds.
- Removing the card mid-run gives `REMOVED`. Pausing the screen mid-run ends the run with no result.
