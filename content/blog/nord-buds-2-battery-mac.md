---
title: "Reverse Engineering OP Nord Buds 2 with AI"
date: 2026-09-04T17:30:00+02:00
draft: false
toc: true
---

> Muse Spark 1.3 used for the reverse engineering process and help with the blog

One thing before we start. I saw that Muse Spark 1.3 had launched with crazy benchmaxx, and it was free to use through [opencode](https://opencode.ai). I decided to test it on this genuinely messy reverse engineering task with no existing documentation.

# Why bother

I use OnePlus Nord Buds 2 which works great with my Android device. The problem is I sometimes forget to charge it. It's quite visible in my phone. I also use it with my macbook, but the battery information is not present. I put a small
[SwiftBar](https://github.com/swiftbar/SwiftBar) plugin in the menu bar that shows
the buds' battery. 

While it worked great, it only showed a single number that's shared to the mac by the device. On Android, HeyMelody app shows all three individual battery levels (including case). On macOS, there's no official support for this. The hardware link is present obviously, but the numbers are locked behind a phone app.

# Existing solutions

In short, nothing specific exists for my hardware. However I did find the following: 
 **[cracked-oneplus-buds](https://github.com/AasheeshLikePanner/cracked-oneplus-buds)**
  reverse-engineered the OPOv1 protocol for the **Nord Buds 3 Pro** over BLE GATT.
  My Nord Buds 2 is listed as likely compatible. But in my testing, it's not.


# Process

## 1. Testing the existing solution

I started with the existing script that I found. I ran the OPOv1 tooling
against my buds and got `FOUND → Connected → Timeout`. Three times. Connected
at the link layer, then nothing. GATT service discovery just never returned.

So I had Spark write me a few BLE scan scripts to check what was even in the
air. Turns out: nothing, most of the time. With the buds connected for audio,
a 15-second scan shows no OnePlus advertisement at all, just Classic
(`HFP/AVRCP/A2DP`). Open the case, still nothing. The buds only advertise LE
during the pairing blink (hold the case button till it goes white), and even
then macOS connects but discovers zero services. I confirmed it from the other
side too: `log stream` during the attempt shows Classic A2DP happily alive
and no ATT traffic whatsoever.

With this finding, the thing that got clear is that my buds didn't talk over BLE, but rather the classic stream.

## 2. Asking the phone for ground truth

I enabled HCI snoop logging on Android, opened HeyMelody, refreshed the battery screen a few times, and pulled
a bug report. (Side note: on modern Android you can't just
`adb pull` the raw `btsnoop_hci.log` without root access. `adb bugreport` works
fine though, and the HCI log rides along under
`FS/data/misc/bluetooth/logs/bt_hci_*.cfa`, plain BTSnoop v1.)

I had Spark chew through the log looking for the OPO framing (`AA …` headers).
79 packets in one window. And as expected, they all are over
**classic SPP/RFCOMM**, and there is no `HELLO`/`REGISTER`/token handshake at
all. The phone just asks, the buds just answer:

```
query: AA 07 00 00 06 01 <SEQ> 00 00        # CAT 06 = battery
reply: AA 0D 00 00 06 81 <SEQ> 06 00 00 02 01 32 02 28
```

The reply echoes the query SEQ, so pairing requests to responses is trivial.
The tail is count-prefixed pairs: `02` entries, `[01]=0x32=50`,
`[02]=0x28=40`. I glanced at HeyMelody: L=50, R=40. That's the whole mapping:
`01` is left, `02` is right. I won't pretend I saw it instantly. The script
found the pattern, which I then confirmed against the app. I was quite surprised that it actually just worked. It did take some hit and trial to get to this point, but Muse held up pretty great.

## 3. Finding the channel on the Mac

With the protocol understood, the Mac still needed to get this information.
I pointed Spark at the buds' SDP records (address from `system_profiler`),
and out of 7 records one jumped out:

```
service "oppointeraction"
uuid128(00 00 11 07 d1 02 11 e1 9b 23 00 02 5b 00 a5 a5)
rfcomm channel 15
```

Same SPP UUID family the HeyMelody decompile references, and channel 15
matched the RFCOMM DLCI from the snoop. While this is great finding, macOS creates no
`/dev/tty.*` for this service (it does for some other buds though), so no
`screen` session. The tool had to open the channel directly through
`IOBluetooth`. A few iterations with Spark on the Swift bridging later, the
first query went out over channel 15 and answered immediately:

```
TX AA 07 00 00 06 01 50 00 00
RX AA 0D 00 00 06 81 50 06 00 00 02 01 32 02 28
RESULT L=50 R=40
```

Same numbers as the phone. 
Also found the buds push unsolicited `04 02` broadcasts on connect
carrying the same triple. Free battery updates without even asking.

## 4. The missing case

Out of the case, the reply holds 2 entries. I docked the buds, re-ran, and it
grew to 3:

```
AA 0F 00 00 06 81 64 08 00 00 03 01 64 02 64 03 5A
```

Three entries: `01:0x64=100`, `02:0x64=100`, `03:0x5A=90`. HeyMelody said the
case was at 90. By elimination, `03` is the case. To be sure it wasn't a fluke
of position, I ran a sweep across category bytes and watched which slots moved:
`03` only ever appears docked, always tracking the case level in the app.

So the parser reads the count `N` and walks `N` `(id, pct)` pairs instead of
assuming fixed offsets. Both docked and undocked replies decode cleanly.

# Result

Two pieces, both in the [repo](https://github.com/ashishsubedi/nord-buds-2-mac-battery):

- `budsbatt` (Swift + IOBluetooth, ~85 lines) opens RFCOMM ch15, sends
  `06 01`, prints `RESULT L=<n> R=<n> [C=<n>]`. About a second per read, and
  `C` only shows up when the buds are docked, because that's the only time
  the buds send it.
- The SwiftBar plugin (`nord_buds_2.5s.py`) tries `budsbatt` first, falls back
  to the old single value if the query fails. The title collapses to `🎧 100%`
  when both buds agree and splits to `🎧 90 · 75` (left first) when they don't.
  Red only below 20%. I don't need panic colors for a half-full bud. The
  dropdown lists Left / Right / Case plus Disconnect and Bluetooth Settings.

I verified it across a charge cycle (50/40, then 50/50, then 100/100 with a
90 case), matching HeyMelody every single time. That's my menu bar now.

# Limitations

I'll be honest about the rough edges, because there are a few:

- One RFCOMM client at a time, so if two things poll at once (my 5 s refresh
  plus a manual run, or HeyMelody on a still-connected phone), the channel is
  busy and the plugin falls back to the single value. I just disconnect the
  buds from Android when I'm at the Mac.
- No case reading when undocked. Out of the case there are 2 entries and
  the Case line hides itself. Nothing to fix here. The buds genuinely don't
  send it.
- Nord Buds 2 only. The Oppo/Realme OPOv1 family probably speaks a similar
  dialect, but I haven't tested any of them.
- No ANC or EQ control yet. The snoop captured those categories too (sets like
  `04 04 …` with the mode byte last), so the obvious next step is replaying a
  captured set and watching what the buds do. That's another post.
