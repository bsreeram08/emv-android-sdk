---
title: Security
nav_order: 9
---

# Security

For how the pieces fit together, see [Architecture and security](architecture.md).

## What the SDK does

- **Clears card data from memory as soon as it is used.** Card commands, card answers, and kernel buffers are overwritten after each step, and the kernel drops its card data when a transaction ends.
- **Never hands card data to your app.** The online request is built and sealed inside the payment core; the app layer gets only a masked card summary and the result. Release builds contain no code path that copies card data out.
- **Keeps card data out of diagnostics.** `NfcStatus`, `NfcEvent`, and `TapTimingReport` hold no tag ID, no card command or answer bytes, and no card fields.
- **Gives safe errors.** A `SafeError` never carries platform exception text.
- **Uses a fresh unpredictable number for each transaction** whenever you configure a profile. A profile cannot pin it.
- **Rejects malformed profiles,** including unknown fields and data after the JSON document.

## What your app must do

- **Protect payment screens** with `FLAG_SECURE`. On Android 13+, also call `setRecentsScreenshotEnabled(false)`.
- **Pin your host keys** in the configuration and serve the host over `https`.
- **Send the online request only over an authenticated, encrypted channel** to your host. Keep it in memory only, and only as long as the call needs it.
- **Load your own profile before going live.** The built-in test profile is refused for any real host (`SYNTHETIC_PROFILE`). See [Accepting a payment](payments.md#1-set-the-amount).
- **Ship release builds as non-debuggable.** Some SDK logging switches itself on only in debuggable builds.
- **Meet your PCI obligations.** Using this SDK does not by itself make your app or system PCI DSS, PCI MPoC, or scheme compliant.

## Reporting a vulnerability

**Do not open a public issue for a security problem.** Use GitHub's private vulnerability reporting on this repository: **Security** tab, then **Report a vulnerability**.
