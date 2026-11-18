# MediaTek MT7925 wakes Windows from S3 when another Wi-Fi client joins a UniFi AP

**Symptom.** A Windows 11 desktop could sleep for half an hour on a quiet network. When another device joined Wi-Fi, the desktop woke about five seconds later. Windows reported `PCI Express Root Port`. The joining device was on 2.4 GHz and the sleeping desktop was on 6 GHz.

**Trigger.** At association, the UniFi U7 Pro forwarded a 20-byte broadcast LLC XID frame onto the desktop's wireless interface. The same client joined twice during one uninterrupted sleep cycle. The AP dropped the first frame and the desktop stayed asleep. The second frame reached 6 GHz. Windows recorded a wake **5.258 seconds later**.

**Workaround.** Drop that specific frame on the AP as it leaves the wireless interface serving the desktop. A separate control confirmed that an intentional magic packet still woke the PC over Wi-Fi. [AP workaround](#ap-workaround) · [Recorded events](evidence/wake-timeline.csv)

## Affected configuration

Confirmed on one MediaTek Wi-Fi 7 MT7925 running the Windows x64 `mtkwecx.sys` driver. The tested package and binary both report **`5.7.0.6342`**. The binary is 2,846,144 bytes and its SHA-256 is `fc7eb2ab3c6c88bed618f616127edcd2f79badae4f3595e176864f556fb30ca2`.

### Does this look like your wake?

Check whether `powercfg /a` lists **Standby (S3)**. On an affected system, the useful clue is the interval between another device joining and the PC waking. A root-port wake by itself gives no packet-level diagnosis. These commands are read-only and can run while the PC is awake. [Microsoft powercfg reference](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options)

```powershell
powercfg /a
powercfg /devicequery wake_armed
powercfg /lastwake

$wifi = Get-NetAdapter -Physical | Where-Object {
    $_.Status -eq 'Up' -and $_.InterfaceDescription -match 'MT7925'
}
$wifi | Format-Table Name, InterfaceDescription, DriverVersion
$wifi | Get-NetAdapterAdvancedProperty | Where-Object {
    $_.RegistryKeyword -match 'Wake|Rekey'
} | Format-Table DisplayName, DisplayValue, RegistryKeyword
```

## Test environment

| Component | Configuration |
|---|---|
| Desktop | Windows 11 25H2 build `26200.9974`, S3 standby, MT7925 associated on 6 GHz |
| Wake settings | Wake on Magic Packet enabled, Wake on Pattern Match disabled, GTK Rekey for WOWLAN enabled |
| Access point | UniFi U7 Pro, firmware `8.7.11`. The 2.4 GHz and 6 GHz SSIDs shared a layer-2 LAN. |
| Joining client | Intel 8265 on 2.4 GHz, no IPv4 address or DHCP client, IPv6 disabled |
| Wake control | Independent wired host with a scheduled Wi-Fi magic packet for the desktop |

## How a 2.4 GHz join reached a sleeping 6 GHz PC

A bridge needs to learn where a newly associated station can be reached. One way to announce it is an 802.2 LLC XID update. The Ethernet destination is broadcast and the source is the **joining station's MAC**. Both AP capture points saw that source address, even though the sleeping PC and the joining station used different radios. The Linux wireless stack documents the frame layout and its forwarding-table purpose in `cfg80211_send_layer2_update`. [Linux implementation](https://linux.googlesource.com/linux/kernel/git/torvalds/linux.git/%2B/a74306e2e676f9775457366fc047a660fbf02f26/net/wireless/util.c)

### Recognizing the frame

| Ethernet field | Captured value |
|---|---|
| Destination | `ff ff ff ff ff ff` |
| Source | MAC of the station that just joined |
| 802.3 payload length | `00 06` |
| Entire LLC payload | `00 01 af 81 01 00` |

The first two payload bytes identify the LLC service access points and mark this as a response. Byte `af` identifies XID. Those six payload bytes plus the 14-byte Ethernet header account for the 20 bytes on the drop counter. The remaining question is why the MT7925 wake path responds to this frame with pattern wake disabled. The network trigger is established. The decision inside the driver or firmware remains unknown. [Field-by-field decode](evidence/README.md#reading-the-packet)

## Evidence

### 1. A join was enough without DHCP

A scan found the 2.4 GHz SSID while the desktop slept through to its backup timer. When the spare client actually associated, the AP bridge and the 6 GHz interface both saw the XID. Windows resumed **5.19 seconds** after the outgoing frame, before the scheduled magic packet. The joining client had no IP address. In another trial the desktop disconnected from Wi-Fi before entering S3 and slept through the spare client's join. [Control trials](evidence/README.md#controls-and-failed-filters)

### 2. Two associations in one sleep cycle

The controlled variable was whether the AP forwarded the test client's XID onto 6 GHz. The AP applied a filter matching the client's source MAC and the exact frame. The client disconnected after its first join, and the AP removed the filter while the desktop was still asleep.

| Seconds since S3 entry | AP or client observation | Desktop |
|---:|---|---|
| 36.607 | First XID on the bridge. The egress filter dropped one 20-byte frame. | Asleep |
| 49.119 | Test client disconnected. | Asleep |
| About 76 | AP removed the filter. | Asleep |
| 112.717 | Same client rejoined. XID present on both bridge and 6 GHz captures. | Asleep |
| 117.975 | Windows recorded a root-port wake. The scheduled magic packet was still about 56 seconds away. | Awake |

A separate attempt attached a drop rule to the bridge firewall. Its counter stayed at zero, the XID reached the wireless interface, and the PC woke 5.03 seconds later. The placement of the rule mattered. [Per-trial timestamps and measurement precision](evidence/README.md)

### 3. Intentional Wake-on-WLAN survived filtering

A temporary rule suppressed broadcasts from the spare client on the desktop's 6 GHz interface. It dropped exactly one frame of 20 bytes. The desktop slept through association and disconnect. At 98.603 seconds the **wired sender** sent a magic packet. At 102.944 seconds Windows recorded the wake, **4.341 seconds after transmission**. [Magic-packet control data](evidence/wake-timeline.csv)

### 4. Checking the GTK rekey hypothesis

A separate MT7922 investigation describes wakes at the AP's group-key rotation interval. On this MT7925, the AP logged the joining client's WPA handshake on 2.4 GHz and no group-key exchange with the sleeping desktop during the join-triggered wake. The Ethernet capture alone could not settle that question. Even when the awake desktop reconnected and `hostapd` logged its four-way handshake, that capture saw zero EAPOL-Key frames. The intervention that changed the outcome here was dropping the XID. [Handshake visibility check](evidence/README.md#group-key-rotation-as-a-possible-trigger) · [MT7922 GTK rekey report](https://github.com/Daiius/mt7922-gtk-rekey-wake)

## AP workaround

On the U7 Pro, open **UniFi Network → UniFi Devices → select the AP → Settings → Debug**, or use the AP's SSH shell. Run `iw dev` to find the interface for the **desktop's SSID and band**. It was `wifi2ap6` on the tested AP. UniFi's built-in multicast and broadcast control is a MAC allowlist that can also affect DHCP and discovery. [UniFi Debug Console](https://help.ui.com/hc/en-us/articles/204909374-Connecting-to-UniFi-with-Debug-Tools-SSH) · [UniFi broadcast controls](https://help.ui.com/hc/en-us/articles/27384925962647-Managing-Broadcast-Traffic-with-UniFi)

First, watch a normal client join while the desktop is awake. This expression is for an Ethernet-format interface (`EN10MB`). Stop the capture with Ctrl+C.

```sh
dev=wifi2ap6  # Set this to the interface serving the desktop.
match='ether broadcast and ether[12:2] = 6 and ether[14:4] = 0x0001af81 and ether[18:2] = 0x0100'
tcpdump -p -i "$dev" -nn -e -tttt -s 20 "$match"
```

In the same shell, check `tc filter show dev "$dev" egress` and choose a free priority. The example uses `43137`. The rule matches the XID from any source MAC, so it covers other clients joining the LAN.

```sh
code=$(tcpdump -i "$dev" -ddd "$match" | tr '\n' ',' | sed 's/,$//')
tc qdisc show dev "$dev" | grep -q 'qdisc clsact' || tc qdisc add dev "$dev" clsact
tc filter add dev "$dev" egress protocol all pref 43137 handle 1 bpf bytecode "$code" flowid 1:1 action drop
tc -s filter show dev "$dev" egress
```

The `dropped` counter should increase on a matching join. The rule affects **every station on that SSID and band at this AP**. A wireless bridge there would miss these updates. ARP, DHCP, IPv6 and ordinary magic packets pass the match. A desktop that roams to a different interface follows that interface's rules. This is a runtime change, so recheck it after AP reboot or Wi-Fi reprovision.

To remove this filter, use the same interface, priority and handle used at installation.

```sh
tc filter del dev "$dev" egress protocol all pref 43137 handle 1 bpf
```

## Reproducing the measurements

1. **Confirm delivery while awake.** Watch the AP bridge and desktop-facing interface during a normal join. Record the frame on both before filtering, then check the bridge capture and the new drop count with the rule active.
2. **Check sleep and intentional wake separately.** Choose a time when the desktop is idle and a known wake method is available. Let the desktop enter sleep, allow a normal client join, then send the intended magic packet at a recorded later time.
3. **Compare the timestamps.** Read **Wake Time** inside Windows Power-Troubleshooter event 1. Windows can write the event several seconds after resume. Compare that field with the outgoing XID or the magic packet. The published CSV uses seconds from S3 entry for each of four trials. [Event timeline](evidence/wake-timeline.csv)

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'System'
    ProviderName = 'Microsoft-Windows-Power-Troubleshooter'
    Id = 1
} -MaxEvents 5 | Format-List TimeCreated, Message
```

## Independent reports

Reports that narrow the scope are useful in either direction. Include the adapter and driver version, Windows build and sleep model, AP model and firmware, and the band and LAN used by each client. The most informative observations are whether the XID reached the desktop-facing interface, how long until Windows woke, and whether a deliberate magic packet still worked. A desktop that stays asleep after **confirmed delivery** is also worth reporting. Relative times are sufficient, and public reports can omit station addresses and SSIDs. [Open an issue](https://github.com/Pyrynot/mt7925-unifi-wowlan-xid-wake/issues)
