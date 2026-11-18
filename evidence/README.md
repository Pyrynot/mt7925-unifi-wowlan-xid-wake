# Evidence for the MT7925 client-join wake

The desktop was associated on 6 GHz and in Windows S3 standby. A separate computer joined the same LAN on 2.4 GHz. Packet captures timestamped the XID on the AP bridge and on the interface serving the desktop. The joining client logged each association and disconnect. A wired host logged when it sent a magic packet. Windows supplied the sleep and wake timestamps in Power-Troubleshooter event 1.

The [CSV](wake-timeline.csv) contains events from four sleep trials. Each trial starts at its own Windows sleep timestamp, omits events after resume, and rounds elapsed time to milliseconds. Bridge and 6 GHz timestamps come from the same AP clock. Client, sender, and Windows timestamps come from different clocks, so the millisecond ordering of client association and frame capture should not be interpreted as processing latency.

## Following one sleep cycle

| Seconds from sleep | Observation |
|---:|---|
| 0.000 | Windows entered S3. The exact-match filter was active on the desktop's 6 GHz interface. For this trial it also matched the test client's source MAC. |
| 36.607 | The AP bridge saw the first join's XID. The outgoing capture recorded none. Over this interval the filter counter rose from one 20-byte drop to two, a delta of one frame. |
| 49.119 | The joining client disconnected. Windows was still asleep. |
| About 76 | The AP removed the filter. Its removal log had one-second timestamp resolution. |
| 112.717 | The same client joined again. The bridge and the outgoing 6 GHz capture both recorded its XID. |
| 117.975 | Windows recorded a wake from the PCI Express Root Port. The independent sender's first magic packet was still about 56 seconds away. |

The interval between the outgoing XID and wake was **5.258 seconds**. The first join is the control for the second. The desktop remained in the same sleep cycle while the client completed both associations, and the AP showed which frame the filter removed.

## Controls and failed filters

| Condition | Observed outcome |
|---|---|
| Quiet network, no scheduled join | The desktop slept for 30 minutes, then woke near the armed backup timer. |
| Spare radio scanned the 2.4 GHz SSID | Its scan saw the SSID while the desktop was asleep. Windows stayed asleep through the scan and resumed near its backup timer, about six minutes after entering S3. |
| Desktop disconnected from Wi-Fi before S3 | The spare client joined while the desktop slept. Windows resumed near its four-minute backup timer. |
| Desktop associated, spare client joined without IP configuration | The ordinary join woke the desktop 5.19 seconds after the outgoing XID. The client needed neither an IPv4 lease nor IPv6 traffic for this wake. |
| Bridge firewall rule enabled | The rule counters stayed at zero. Both captures saw the XID and Windows woke 5.03 seconds later. An installed rule with zero hits gave no evidence of filtering. |
| Source-MAC broadcast filter on 6 GHz | The filter dropped one frame of 20 bytes. The bridge saw the XID, the outgoing capture saw none, and Windows stayed asleep through the join and disconnect. Windows woke 4.34 seconds after the wired host sent its magic packet. |

For the timer controls, Windows described the source as unknown with a possible timer owner. The recorded resume times and independently scheduled magic packets distinguish those controls from a wake at association.

## Reading the packet

The captured Ethernet frame exposes all the fields used by the workaround. Offsets below are zero-based. The capture did not include a frame check sequence.

| Byte offset | Value | Meaning |
|---:|---|---|
| 0–5 | `ff ff ff ff ff ff` | Broadcast destination |
| 6–11 | Joining station's MAC, omitted | Address whose location bridges can learn |
| 12–13 | `00 06` | 802.3 payload length in bytes |
| 14–15 | `00 01` | Null destination and source service access points, with the response bit set on the source |
| 16 | `af` | LLC XID control byte |
| 17–19 | `81 01 00` | XID format identifier, Type 1 LLC, receive window |

The filter requires the broadcast destination, length, and all six payload bytes. The deployed rule accepts any source MAC so it also matches updates for other joining clients. It discards the copy about to leave this wireless interface. The bridge capture continued to see the original update. [Linux layer-2 update implementation](https://linux.googlesource.com/linux/kernel/git/torvalds/linux.git/%2B/a74306e2e676f9775457366fc047a660fbf02f26/net/wireless/util.c)

## Group-key rotation as a possible trigger

A separate MT7922 report linked unwanted wakes to group-key rotation. To check which instruments could see a known handshake on the MT7925, the **awake** desktop briefly disconnected and rejoined. The AP's `hostapd` log recorded the desktop's four-way handshake. The simultaneous Ethernet-format capture on that interface saw zero EAPOL-Key frames. That capture had a blind spot even during a known exchange.

During the unfiltered join trial, `hostapd` recorded the **joining client's** handshake on 2.4 GHz. Its log contained no group-key exchange with the **sleeping desktop** on 6 GHz during that wake. The two-join trial isolated the XID by changing its delivery while the desktop remained asleep. A machine waking at a fixed group-key rotation interval could have a different trigger. [Related MT7922 report](https://github.com/Daiius/mt7922-gtk-rekey-wake)
