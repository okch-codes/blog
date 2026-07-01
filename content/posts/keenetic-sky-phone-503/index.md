---
title: "My Sky phone line kept dying for days. The fix was one SIP flag."
date: 2026-07-01T08:00:00+02:00
draft: false
tags: ["keenetic", "sky-wifi", "voip", "sip", "100rel", "ipv6"]
categories: ["notes"]
summary: "Sky Phone on my Keenetic would drop and throw a 503, then stay dead for days. The tempting fix was a nightly reboot. The real fix was a single SIP option: 100rel."
ShowToc: true
TocOpen: false
---

Same setup as [the Wi-Fi story](../instagram-slow-wifi/): Sky Wifi FTTH in
Italy, a Keenetic Titan (KN-1811) plugged straight into the ONT and speaking
MAP-T natively. This time the problem wasn't the internet — it was the phone.
Sky Phone (Sky's VoIP line) would work fine, then drop with a **503**, and
then stay dead. Not for minutes. Sometimes for *days*, until a reboot or a
DHCP renewal happened to knock it back to life.

The obvious instinct with a phone line that "just needs a kick" is to
schedule the kick. That was my plan. It turned out to be the wrong fix for
the right symptom — the actual cause was one SIP option that Keenetic doesn't
enable by default for Sky's Italian VoIP.

Here's the whole path.

## The error: 503, "No answer record in the DNS response"

Keenetic's IP Telephony module doesn't register to a fixed IP. It registers
to Sky's SIP proxy by *name* — something like `voip.glb.it.isp.sky` — and to
get there it does two lookups in sequence:

1. A DNS **SRV** lookup to find the SIP service host.
2. An **AAAA** (IPv6) lookup to resolve that host, because Sky's VoIP backend
   (the SBC) is reached over IPv6.

If either lookup fails at the exact moment the router tries to re-register,
Keenetic eventually gives up with `503 — No answer record in the DNS
response`, and the line stays down until registration succeeds again.

So the first hypothesis was the same family of suspect as the Instagram
post: **IPv6**. On a MAP-T network, VoIP leans on IPv6 even when ordinary
browsing is happily riding IPv4. If the MAP-T session flaps or the ISP
profile re-negotiates badly, AAAA resolution for the SBC host fails, the
phone module can't recover on its own, and — because some firmware versions
don't retry aggressively — it sits broken until something external kicks it.
That "no automatic recovery" part is what turns a momentary hiccup into a
multi-day outage.

If you want to catch it in the act, resolve the Sky names directly from a
machine behind the router, during a working window and (if you can) during a
failure:

```bash
nslookup -type=SRV _sip._udp.voip.glb.it.isp.sky
nslookup -type=AAAA <hostname the SRV record returns>
```

If those fail while `google.com` resolves fine, you've confirmed it's
specifically the Sky SBC name failing — which is exactly the evidence support
wants.

## The tempting fix: bounce just the phone, nightly

My first plan was a workaround, not a cure: if the line only needs a nudge to
re-resolve and re-register, schedule the nudge before it gets stuck.

Keenetic's scheduler isn't reboot-specific. You define a named time window,
then attach a command to fire in that window — the same pattern people use for
a nightly router reboot. And there *is* a narrower target than a full reboot:
the Phone Station toggle deactivates just the telephone exchange
(deregistering every line, powering down the handsets) without touching Wi-Fi
or routing.

Honest caveat, and it's the same discipline as diagnosing with data: I could
confirm that toggle exists and that the scheduler can drive arbitrary
commands, but I could **not** verify Keenetic's exact CLI keyword for that
specific switch. I'm not going to hand myself a guessed command for something
that runs unattended on my only phone line. The way to nail it down is to let
the CLI tell you:

```
# SSH/Telnet in, enter config mode, then:
nvox sip 0     # your line id, then press Tab to list every valid keyword
```

Tab-completion lists the real options (`keep-alive-extended`, `deny-pickup`,
and friends), so you toggle the right one, confirm in the Web UI that the line
drops to *Disabled* then returns to *Registered*, and only then wrap it in a
schedule block.

The fully-confirmed, blunter fallback is a scheduled full reboot at a quiet
hour:

```
schedule nightlyreboot
 action start 0 4 *
 action stop 1 4 *
exit
system reboot schedule nightlyreboot
system configuration save
exit
```

It works, but it's a symptom mask — a nightly blip on Wi-Fi *and* phone to
paper over a bug I hadn't actually understood yet. Before building anything
around the symptom, I went looking for the cause. That's where it got
interesting.

## The real fix: 100rel

Sky's Italian VoIP service has a specific set of requirements that Keenetic
does **not** all enable by default: support for `100rel` (reliable provisional
responses), both G.711 and G.729 codecs, 20 ms packetization, a registration
expiry of 3600 s, voice activity detection off, a specific conference address,
and 802.1p QoS tagging to prioritize voice over data on the fiber.

The one that jumps out is `100rel`. Missing it produces exactly this shape of
bug: **registration looks fine, but call handling silently breaks**, and over
time that manifests as the line going dead for stretches. Keenetic's own fix
is two lines from the CLI:

```
nvox sip-common 100rel
copy running-config startup-config
```

The first turns it on; the second saves it so it survives a reboot. While
you're in there it's worth checking the rest of that list against your line
config — codecs, packet timing, registration timeout — because a mismatch on
any of them produces the same flaky, hard-to-pin-down behavior.

But *why* does a single flag matter this much? That's the part worth
understanding.

## What is 100rel, actually?

`100rel` is a SIP extension ([RFC 3262](https://www.rfc-editor.org/rfc/rfc3262))
whose name is short for "100% reliable." It makes certain in-progress call
messages reliable — the same guarantee the "call answered" message already
has.

SIP responses come in two flavors:

- **Final responses** (`200 OK`, `4xx` errors, …) are always reliable. If the
  other side doesn't acknowledge one, the sender keeps retransmitting until it
  does.
- **Provisional responses** (`1xx` codes like `180 Ringing` or
  `183 Session Progress`) are normally fire-and-forget. Sent once, no required
  acknowledgment. If the UDP packet carrying one is lost, it's simply gone.

Usually that's harmless — a dropped `180 Ringing` doesn't matter, the call
connects or fails regardless. But one provisional response carries real
payload: `183 Session Progress` often includes **SDP**, the media negotiation
used for *early media* — the actual network ringback tone, an announcement, or
precise codec setup *before* the call is even answered. If that packet
silently drops, the caller gets dead air, or the session gets wedged
half-finished.

`100rel` closes that gap by adding:

- a **PRACK** method — an ACK, but for provisional responses instead of final
  ones;
- **RSeq / RAck** headers so each PRACK matches the specific provisional
  response it confirms.

With `100rel` on, that important provisional response is retransmitted until
explicitly acknowledged, instead of hoped-for.

```
Caller (Keenetic)                         Sky SBC
      |                                      |
      |  INVITE  ----------------------->    |
      |                                      |
      |  <-----------------  183 Session Progress (SDP)   [reliable now]
      |                                      |
      |  PRACK  ------------------------>    |   <- the step 100rel adds
      |  <-----------------------  200 OK (PRACK)          |
      |                                      |
      |  <-----------------------  200 OK (INVITE)         |
      |  ACK  -------------------------->    |
      |                                      |
```

Without `100rel`, that `183` is sent once over UDP and hoped for. With it,
Keenetic *must* send `PRACK` and Sky *must* confirm before setup proceeds —
the same reliability the final `200 OK` already enjoys.

## Why it matches the symptom

This is the satisfying part. **Registration is a separate, simpler
transaction that keeps succeeding either way** — which is precisely why the
line kept showing up as fine. But if Sky's network expects the PRACK handshake
during call setup and Keenetic never sends it, the parts of the flow built on
`100rel` stall or behave unpredictably: call setup hangs, sessions get stuck —
without ever touching registration.

"Line shows up but the actual phone service quietly breaks for days" is
exactly what a missing `100rel` looks like. The 503/DNS story was real and
worth ruling out, but it was the loud failure mode; the quiet one was a
protocol option nobody enabled.

## Takeaways

1. **"Registered" is not "working."** Registration and call setup are
   different SIP transactions. A line that shows *Registered* can still be
   broken if the call-setup handshake it depends on isn't happening.
2. **Reach for the root cause before the cron job.** A nightly bounce would
   have "fixed" this forever while hiding it forever. Scheduled reboots are a
   safety net, not a diagnosis.
3. **Default configs aren't carrier-tuned.** Sky Italia has a specific VoIP
   profile — `100rel`, codecs, packetization, registration expiry, QoS.
   Matching the carrier's expectations matters more than any single toggle.
4. **Don't automate a guessed command.** I could confirm the Phone Station
   toggle and the scheduler existed, but not the exact CLI keyword — so I let
   Tab-completion tell me rather than shipping a guess into unattended
   automation on my only phone line.

The DNS/IPv6 angle wasn't crazy — but this time the story was one flag deep in
the SIP stack.
