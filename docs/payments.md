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
- A problem before the request is sent ends as `HostFailure` (for example `TRANSPORT` or `NOT_CONFIGURED`) and never as an approval.
- Release builds only talk to an `https` host whose certified key matches the configured gateway (and the pins, if any).

### When the host's answer is lost

Every sale gets a transaction id and a STAN (the system trace audit number) before the card is read. Both are on the phone's storage before the request leaves; the STAN is the one in the authorisation message. Both come back on the result as `transactionId` and `stan`: print them on the receipt.

If the answer does not arrive (a timeout, a dropped connection, a server error, or an answer that does not check out), the SDK does not guess. In the same sale it asks your host what became of that transaction, a few times over up to about 30 seconds, and reports:

- `Complete` with `APPROVED` or `DECLINED` when the host knows the outcome;
- `HostFailure` with `REVERSED` when the outcome stayed unknown and the SDK reversed the sale: nothing was charged; take the payment again;
- `HostFailure` with `UNRESOLVED` when not even the reversal could be confirmed. The card may have been charged. Don't take the payment again until it is resolved: the SDK keeps sending the reversal each time it starts.

The SDK never reports a failure for a sale the host approved and you were shown. An approval you were never shown complete does not stand: if the app was closed mid-sale and the host had approved, the SDK reverses that sale at the next start. The record it keeps holds no card data, only the ids, amount, currency, times and state.

Sales left open by a lost reversal, or by the app being closed mid-sale, are resumed in the background when the reader starts (the first `onResume`). Set a listener to hear how they end:

```kotlin
controller.transactionListener = TransactionListener { record ->
    // record.transactionId, record.stan, record.amount, record.currency,
    // record.state: RESOLVED (record.decision "approved" or "declined"), REVERSED or UNRESOLVED;
    // record.approvedThenReversed: the host had approved it, the sale was interrupted, and the
    // SDK reversed it (no charge stands once state is REVERSED)
    reconcile(record)
}
```

`controller.transactions()` lists the recorded sales (it reads storage: call it off the main thread). A sale still open after 24 hours is marked `UNRESOLVED` and no longer sent; reconcile it with your acquirer.

## 4. The result

`AcceptState.Complete` carries:

- `outcome`: `APPROVED` or `DECLINED`;
- `responseCode` and `body`: from your host;
- `recordsRead`;
- `dataRecordDigest`: a hash you can use as a receipt reference. It is not card data;
- `transactionId` and `stan`: the sale's identity with your host.

After any result, the card connection ends and the controller returns to `WaitingForTag`, so the next sale always starts with a new tap. If the card read fails (for example the card is removed too early), the reader stays on: ask the customer to tap again.

## Cancelling

```kotlin
val cancelled = controller.cancelActiveExchange(sessionId)
```

This ends only the card run in progress for that `sessionId`. It returns `false` when nothing is running, for example when the run has already finished. A cancelled run reports `Failure(CANCELLED)`, and nothing from it can complete a later sale.

## Threading and timeouts

- Each controller runs one card exchange at a time. A second `runAccept` while one is running fails with `BUSY`.
- The host call runs on its own thread: the next card can be tapped and read while the host is still answering the previous sale. Sales reach the host one after another.
- Connecting to a card times out after 2 seconds. A single card command times out after about 5 seconds.
- Removing the card mid-run gives `REMOVED`. Pausing the screen mid-run ends the run with no result.
