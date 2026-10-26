---
title: Accepting a payment
nav_order: 4
---

# Accepting a payment

## 1. Set the amount

Load your configuration once, as described in [Configuration](configuration.md). Then set the amount before each sale:

```kotlin
EmvSdk.prepareSale(amountMinor)
```

A profile you configure always draws a fresh unpredictable number for each transaction.

**Never send the default profile's results to a real host.** If you never call `EmvSdk.load`, or after `AcceptProfile.configure("")`, the SDK uses its built-in synthetic test profile. That profile pins the unpredictable number so tests are repeatable, which means a cryptogram made with it could be replayed. Use it only with a simulated host.

## 2. Run the sale on the connected card

When `SessionState.Ready` arrives, call `runAccept`:

```kotlin
controller.runAccept(sessionId, host) { state ->
    when (state) {
        is AcceptState.Exchanging -> showProgress(state.stage)       // PPSE, SELECT, GPO, RECORDS, ...
        is AcceptState.OnlineRequest -> showSendingToHost()           // the success tone has played; the card can be removed
        is AcceptState.Complete -> showResult(state.outcome, state.responseCode)
        is AcceptState.HostFailure -> showHostProblem(state.reason)
        is AcceptState.Failure -> showError(state.error, state.outcome)
    }
}
```

The card part usually takes a fraction of a second. Ask the customer to hold the card still until you receive `OnlineRequest` or a failure.

## 3. Authorise with your host

`HostClient` is a single function. The SDK calls it on its own background thread, so it can block on the network:

```kotlin
val host = HostClient { request ->
    val contract = request.contract()
    // contract.scheme, applicationId, pan, expiry, amount, currency, cryptogram
    // request.paymentRequestJson: the full kernel-built request (ISO 8583 0100 hex, DE 55, all tags)
    when (val answer = myGateway.authorize(request.paymentRequestJson)) {
        is Ok -> HostResult.Approved(answer.responseCode, answer.body)
        is Refused -> HostResult.Declined(answer.responseCode, answer.body)
        is Timeout -> HostResult.Failed(HostFailureReason.TIMEOUT)
    }
}
```

- The kernel builds the request, and your code does not assemble card fields. `OnlineRequestContract` lists the fields you can depend on across versions.
- The `toString()` of the request, the contract, and every `HostResult` is redacted, so logging one by mistake does not leak card data.
- Use `HostResult.Failed` for transport problems. The SDK then reports `AcceptState.HostFailure` and never approves.

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
