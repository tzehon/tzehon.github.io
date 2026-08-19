---
layout: default
title: Wardrobe Stylist Privacy Policy
description: How Wardrobe Stylist handles local, Gmail, AI, and security data
permalink: /wardrobe/privacy/
published: false
---

# Wardrobe Stylist Privacy Policy

> **Unpublished release draft.** This policy is not yet effective. It is intentionally excluded
> from the live GitHub Pages build until the production evidence listed under **Before
> publication** is complete.

**Privacy contact:** [contact@tth.dev](mailto:contact@tth.dev)

**Support response target:** Within two business days.

Wardrobe Stylist helps you build a wardrobe catalog and get outfit suggestions. You can add items
manually or from photos without connecting Google. If you connect Gmail, the app uses read-only
access to look for purchase receipts. Gmail connection, AI processing, background receipt import,
and reminders are optional.

## Information stored on your device

The app stores wardrobe item details, photos you choose or capture, purchase details you approve,
cached outfit recommendations, wear history, feature preferences, consent records, and sync state
on your device. This information remains until you delete it or remove the app. The final policy
will state the verified Apple backup and restore behavior for the SwiftData store, external photo
files, preferences, and credential stores; that evidence is not complete yet.

## Google and Gmail information

If you connect Gmail, the app requests only the
`https://www.googleapis.com/auth/gmail.readonly` scope. This permits reading Gmail but not sending,
modifying, labeling, trashing, or deleting messages. On-device code searches for likely purchase
receipts and extracts deterministic structured fields where possible.

When a receipt needs cloud analysis, the app sends the developer backend a validated sender
domain, a sanitized subject, and either bounded structured product fields or selected, redacted
product lines. The current compatibility envelope also sends a Gmail message identifier to the
backend for response correlation; the backend removes that identifier before sending minimized
product context to Anthropic. Full sender addresses and raw message bodies are not sent to
Anthropic.

## Wardrobe information used for AI styling

If you consent to AI styling, the app may send compact wardrobe attributes—such as internal item
ID, name, category, brand, colors, and material—plus recent-wear identifiers, bounded per-item
rating summaries, and an occasion you provide to the developer backend and Anthropic. Rating free
text, wear dates, wardrobe photos, and purchase metadata are not included in styling requests.

## Technical and security information

The backend necessarily receives network information such as an IP address and request timing
while servicing a request. The developer authentication database stores keyed HMAC rate-limit
subjects rather than raw IP addresses. Application security events are designed to omit IP
addresses, installation and key identifiers, credentials, request content, and model content.
Fly.io has confirmed that its customer-visible proxy/platform error records can nevertheless
include paths, request IDs, and sometimes client IP, and that separate provider operational or
abuse-prevention logs can contain source IP. The customer-visible stream lasts seven days and
cannot be shortened per app; the separate provider-internal in-service retention is undisclosed
and not customer-configurable. These retained technical records support app functionality, network
delivery, abuse prevention, security, reliability, and diagnostics. They may be associated with an
app installation or request, and are not used by the developer for advertising or cross-company
tracking.

Anonymous App Attest security metadata can include key and challenge IDs, one-time challenge
secrets, a verified public key, an anonymous installation ID, assertion counter, App ID and
environment, optional Apple-signed validation-category and app-build values, an opaque Apple
attestation receipt, session IDs and one-way bearer-token hashes, and bounded rate-window scopes,
counts, timestamps, and keyed HMAC subject hashes. App Attest identifies one app installation; it
is not a human account and does not survive reinstall, migration, or restore. The App Attest
private key remains in the Secure Enclave and raw bearer tokens are not stored by the backend.

## How information is used

The developer uses information only to provide features you request: importing wardrobe items
from receipts, generating outfit suggestions, preventing excessive repeats, securing and
operating the service, diagnosing failures, and complying with law. The developer does not use
Gmail or technical/security data for advertising, data brokerage, credit or lending decisions,
advertising profiles, or tracking across other companies' apps and websites.
Provider handling remains governed by the provider terms and configurations that must be verified
before this draft is published.

Wardrobe Stylist is designed to follow the Google API Services User Data Policy, including its
Limited Use requirements. This statement will be finalized only after Google accepts the
production architecture and disclosures.

## Processing providers

- **[Google](https://policies.google.com/privacy)** provides authentication and the read-only
  Gmail API when you connect Gmail.
- **[Fly.io](https://fly.io/legal/privacy-policy/)** hosts the developer-controlled backend and
  encrypted authentication volume.
- **[Anthropic](https://www.anthropic.com/legal/privacy)** processes minimized receipt or wardrobe
  inputs to return structured items or outfit suggestions when you explicitly use an AI feature.
- **[Apple](https://www.apple.com/legal/privacy/)** provides iOS, notifications, optional
  device-backup behavior, App Attest, and App Store distribution.
- **[GitHub Pages](https://docs.github.com/en/site-policy/privacy-policies/github-general-privacy-statement)**
  hosts this public support and privacy documentation and necessarily processes website-request
  network data when someone visits these pages.

The final policy will identify the verified processing locations, subprocessors, contractual
retention, training/model-improvement and human-access settings, deletion behavior, and applicable
international-transfer mechanism for the selected distribution markets.

## Retention

The developer backend does not persist Gmail receipt text, wardrobe payloads, or model request and
response content after a request completes. It retains only the minimum authentication, security,
and abuse-prevention records required to operate the public service.

The implemented live-store limits are:

- one-time challenges are valid for 5 minutes and purged no later than 70 minutes after issue;
- session-token hashes are valid for 15 minutes and purged no later than 20 minutes after issue;
- keyed rate-limit subjects are purged no later than the applicable window plus 5 minutes;
- active installation metadata is removed after 90 days without successful authenticated use;
- revoked installation metadata is removed after 30 days; and
- an App-Attest-verified deletion request synchronously removes that installation's live security
  record and sessions, within the policy's 24-hour maximum.

The authentication volume is encrypted and configured for rolling 14-day snapshots. Fly.io says a
snapshot then disappears from the customer listing, but does not disclose all-copy purge timing.
Fly.io also says the customer-visible log stream is retained for a fixed seven days and cannot be
shortened per app; its separate provider operational/abuse-log in-service retention is undisclosed
and has no customer-enforceable hard maximum. These hosting records are separate from the live
server record and are not removed immediately by the in-app deletion action. The separate 24-hour
live server-deletion deadline remains unchanged.

## Your choices and controls

You can use a local wardrobe without Gmail, decline or withdraw receipt-analysis and styling
consent, disable background import and reminders, sign out, disconnect Gmail, and delete local
wardrobe data. You can separately use **Settings → Privacy & Data → Delete Server Security
Data** to prove installation control with App Attest and delete that installation's live
anonymous server record and sessions. That action does not delete the local wardrobe or
disconnect Google; future remote-AI use enrolls a new anonymous identity.

You can also review or revoke Google access through
[Google Account third-party connections](https://myaccount.google.com/connections).

For privacy, access, correction, deletion, or security questions, email
[contact@tth.dev](mailto:contact@tth.dev). Never send credentials, App Attest material, Gmail
content, receipt text, or wardrobe photos.

## Security

Security measures include HTTPS in transit, platform credential storage, least-privilege
read-only Gmail access, schema validation, anonymous App Attest backend authorization, rate
limits, dependency review, and automated regression checks. No method of storage or transmission
is completely secure.

## Children

Wardrobe Stylist is not directed to children. The final statement will be reconciled with the App
Store age rating and selected distribution regions before publication.

## Changes

The effective date will be added when this policy is published. Material changes will be reflected
here. When a change affects an optional AI data flow, the app will require consent to the updated
notice before that flow resumes.

## Before publication

The following release evidence or final publication decisions remain required:

- observation of actual 14-day snapshot-list disappearance and an isolated restore/deletion
  rehearsal;
- final production request captures and processor-retention review;
- server-deletion evidence from the processed TestFlight build;
- verified Apple backup and restore behavior for every local store;
- final App Store Connect declarations for retained Device ID, Other Diagnostic Data, and Product
  Interaction used for App Functionality, conservatively linked and not used for tracking;
- direct provider terms, processing locations, subprocessors, retention, training, human access,
  deletion, and international-transfer mechanisms;
- a decision whether to request, review, and sign Fly.io's optional DPA; the account currently has
  no active DPA, so its exact agreement/version and applicability are not yet evidenced;
- applicable user rights, request-verification and response procedures, any required postal
  address, and the final support URL; and
- the effective date, legal owner wording, target markets, and final age-rating reconciliation.

Until those items close and `published: false` is removed, this file is review material rather
than the public Wardrobe Stylist privacy policy.
