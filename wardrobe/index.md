---
layout: default
title: Wardrobe Stylist
description: Support and privacy controls for the Wardrobe Stylist iOS app
permalink: /wardrobe/
published: false
---

# Wardrobe Stylist

> **Unpublished release draft.** This page is intentionally excluded from the live GitHub Pages
> build until its response target and production evidence are complete.

Wardrobe Stylist is a personal, local-first iOS fashion-stylist app. You can catalog clothing from
photos, optionally import purchase information through read-only Gmail access, and request outfit
suggestions. Manual and photo wardrobe features work without connecting Google.

## Support

Email [contact@tth.dev](mailto:contact@tth.dev).

When reporting a problem, include the Wardrobe Stylist version and build, iOS version, device
model, what you expected, and what happened. Do not send receipt contents, Gmail messages, OAuth
tokens, backend credentials, App Attest material, or private wardrobe photos.

**Release blocker:** Choose, publish, and rehearse a business-day response target before making
this page live.

## Common questions

### Do I need Gmail?

No. Manual and photo wardrobe features work locally without Google. Gmail is an optional,
read-only receipt importer.

### Can Wardrobe Stylist modify my email?

No. It requests only Google's `gmail.readonly` permission and contains no Gmail write operation.

### What information is sent for AI processing?

Receipt extraction and outfit recommendations can send minimized information to the developer
backend and Anthropic. The app asks separately before each type of AI processing and keeps those
features off until you consent.

### How do I stop background import or reminders?

Open **Settings → Connected Features** and turn off the relevant control. Background import is
opportunistic; iOS does not guarantee an exact schedule. Reminders are local notifications and can
also be controlled in iOS Settings.

### How do I delete my local wardrobe?

Open **Settings → Privacy & Data → Delete Local Data**. This removes local items, photos,
outfits, wear history, receipt-sync history, cached images, recommendations, and related choices
from this device. It does not delete email, revoke Google access, or delete server security
metadata. Use **Settings → Connected Features → Disconnect Google** separately if you also want
to revoke the Gmail authorization, and use the separate server-security-data control described
below when you want to remove that installation's backend identity.

### What is the difference between Sign out and Disconnect Google?

Sign out ends the app's local Google session. Disconnect Google also revokes Wardrobe Stylist's
Google authorization so it cannot access Gmail again unless you reconnect. Neither action changes
or deletes Gmail messages.

### How do I delete server security data?

Wardrobe Stylist does not create a Wardrobe user account for you and does not use your Google
identity for backend access. Its server security record belongs to one anonymous App Attest
installation and is separate from the wardrobe stored on your iPhone.

Before uninstalling, open **Settings → Privacy & Data → Delete Server Security Data**. The app
uses a fresh App Attest proof to delete that installation's live server identity and active AI
sessions. This action does not delete the wardrobe on your iPhone or disconnect Google. Remote AI
creates a new anonymous identity the next time you use it.

Reinstalling creates a new identity but does not prove that the old record was deleted. If the
original installation is unavailable, email [contact@tth.dev](mailto:contact@tth.dev) for general
guidance. Support cannot safely identify an unlinked anonymous record and will never ask for a
token, key, attestation object, Gmail content, or wardrobe photo. The server's inactivity policy
removes an unused installation after 90 days; encrypted rolling snapshots are separately subject
to the retention described in the [privacy policy](/wardrobe/privacy/).

### How do I revoke Google access outside the app?

Use [Google Account third-party connections](https://myaccount.google.com/connections), select
Wardrobe Stylist, and remove access.

## Privacy

Read the [Wardrobe Stylist privacy policy](/wardrobe/privacy/).
