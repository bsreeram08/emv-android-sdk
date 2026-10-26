---
title: Support
nav_order: 8
---

# Support

Report bugs and ask questions in the [issue tracker](https://github.com/bsreeram08/emv-android-sdk/issues). The issue tracker is public.

## Never post card data

Do not paste any of these into an issue:

- a PAN, expiry date, track data, cryptogram, or the online request;
- card command or answer (APDU) logs;
- photos of cards;
- keys, certificates, or host credentials.

The SDK's diagnostics are designed to be safe to share, so use these instead.

## What to include in a bug report

1. **Phone:** the model and Android version.
2. **SDK version:** from your delivery.
3. **Readiness:** `NfcDiagnostics.status(context)`, printed as text.
4. **The tap's events** from your `diagnosticsListener`, and the `Tap` JSON if there is one.
5. **What you saw:** the `SafeError` or `AcceptOutcome`, and what the screen showed.
6. **Card type:** for example "Visa debit, contactless". Not the card number.
7. **How often it happens,** for example 3 taps out of 10, and whether other cards behave the same.

## Security problems

Report security problems privately, never as an issue. See [Security](security.md#reporting-a-vulnerability).
