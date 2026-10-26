---
title: Overview
nav_order: 1
---

# EMV Android SDK

The SDK lets an Android app accept contactless EMV cards on the phone's NFC reader, with no separate card terminal. It covers the whole card side of a sale: it reads the card over NFC, runs the EMV kernel, builds the online request for your host, and finishes the transaction with the host's answer.

## Status

The SDK is **pre-release**. Before you rely on it, know what it does not have yet:

- **Certification:** no EMVCo Level 2 or scheme certification.
- **Physical testing:** no recorded physical test batch on a named device.
- **Card schemes:** the Visa and Mastercard paths run end to end. Kernel 8 (EMV C-8) is in progress, and a real Kernel 8 card does not complete a transaction yet.
- **Acquirer:** no live acquirer is connected. You bring the host.

Do not put it in front of real customers until the release notes for your licensed version say otherwise.

## Access

The SDK is commercial software. The library (AAR) is not published on this site or in any public repository. Licensed customers download it through authenticated, signed delivery. To ask about a licence, open an [access request](https://github.com/bsreeram08/emv-android-sdk/issues/new?template=access.yml).

This site and the [issue tracker](https://github.com/bsreeram08/emv-android-sdk/issues) are public, so you can review the API before you license it.

## What is in the SDK

| Part | What it does |
|---|---|
| `NfcSessionController` | Owns reader mode, the card connection, timeouts, and one card run at a time. |
| `runAccept` + `HostClient` | Runs a contactless sale. Your `HostClient` receives the kernel-built online request and returns approved, declined, or failed. |
| `AcceptProfile` | Sets the terminal profile: amounts, limits, the AIDs it accepts, and merchant and terminal data. |
| `NfcDiagnostics` and `NfcEvent` | Report NFC readiness and an event for each tap, for support screens. They carry no card data. |
| `SafeError` and `AcceptOutcome` | Name every failure, without platform or card text. |

## Pages

- [Getting started](getting-started.md): requirements, manifest, and lifecycle.
- [Accepting a payment](payments.md): the sale flow, the host, and cancelling.
- [Terminal profile](profile.md): the profile JSON.
- [NFC diagnostics](diagnostics.md): readiness, tap events, timing, and troubleshooting.
- [Errors and outcomes](errors.md): what each failure means and what to show.
- [Security](security.md): what the SDK protects and what your app must do.
- [Support](support.md): how to report a problem.
