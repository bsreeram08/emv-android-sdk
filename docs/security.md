---
title: Security
nav_order: 8
---

# Security

## What the SDK does

- **Clears card data from memory as soon as it is used.** Card commands, card answers, and kernel buffers are overwritten after each step, and the kernel drops its card data when a transaction ends.
- **Redacts card data from logs.** `toString()` on `AcceptOnlineRequest`, `OnlineRequestContract`, `HostResult`, and `AcceptState.Complete` and `HostFailure` leaves out the PAN, expiry, track data, cryptogram, and issuer data.
- **Keeps card data out of diagnostics.** `NfcStatus`, `NfcEvent`, and `TapTimingReport` hold no tag ID, no card command or answer bytes, and no card fields.
- **Gives safe errors.** A `SafeError` never carries platform exception text.
- **Uses a fresh unpredictable number for each transaction** whenever you configure a profile. A profile cannot pin it.
- **Rejects malformed profiles,** including unknown fields and data after the JSON document.

## What your app must do

- **Protect payment screens** with `FLAG_SECURE`. On Android 13+, also call `setRecentsScreenshotEnabled(false)`.
- **Don't log** the fields of `OnlineRequestContract` or `paymentRequestJson`. They are card data.
- **Send the online request only over an authenticated, encrypted channel** to your host. Keep it in memory only, and only as long as the call needs it.
- **Never use the built-in synthetic profile with a real host.** See [Accepting a payment](payments.md#1-set-the-profile).
- **Ship release builds as non-debuggable.** Some SDK logging switches itself on only in debuggable builds.
- **Meet your PCI obligations.** Using this SDK does not by itself make your app or system PCI DSS, PCI MPoC, or scheme compliant.

## Reporting a vulnerability

**Do not open a public issue for a security problem.** Use GitHub's private vulnerability reporting on this repository: **Security** tab, then **Report a vulnerability**.
