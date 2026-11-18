# MediaTek MT7925 wakes Windows from sleep when another Wi-Fi client joins a UniFi AP

A Windows 11 desktop connected to a UniFi U7 Pro on 6 GHz woke from S3 about five seconds after an unrelated device joined on 2.4 GHz. Both SSIDs shared a LAN. On an ordinary join, the AP forwarded a **broadcast LLC XID layer-2 update** onto the desktop's Wi-Fi interface. Dropping that frame at the AP let the desktop sleep through the join and still wake on an intentional Wi-Fi magic packet (WoWLAN).

In **one uninterrupted sleep cycle**, the same client joined twice. The AP dropped the first join's frame, then removed the filter while the desktop slept. The second frame reached 6 GHz, and Windows recorded a wake 5.26 seconds later. [Experiment notes and controls](evidence/README.md) · [Event timeline (CSV)](evidence/wake-timeline.csv)

## Does this look like your wake?

The clue on this machine was the timing. The PC could sleep for half an hour on a quiet network. An unrelated device reconnecting to Wi-Fi could wake it within seconds, even though **Wake on Pattern Match** was disabled. The devices used different bands and SSIDs on the same LAN.

These read-only PowerShell commands work while the PC is awake. The first shows whether Windows offers S3 (`Standby (S3)`). The next two show devices armed to wake Windows and what Windows attributed the last wake to. Open an elevated PowerShell if access is denied. [Microsoft powercfg reference](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options)

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

Get-WinEvent -FilterHashtable @{
    LogName = 'System'
    ProviderName = 'Microsoft-Windows-Power-Troubleshooter'
    Id = 1
} -MaxEvents 5 | Format-List TimeCreated, Message
```

Read **Wake Time** inside the event message when comparing with an AP log. Windows wrote the event several seconds after the recorded resume. **PCI Express Root Port** identifies the hardware path. To identify the traffic, run the passive capture below while the desktop is awake and watch a normal client join. Seeing an XID confirms that the AP forwards the frame on that interface. Linking it to a wake requires the sleep comparison. [Recognizing the frame](#recognizing-the-frame)

## Hardware and settings

| Component | Tested configuration |
|---|---|
| Desktop Wi-Fi | MediaTek Wi-Fi 7 MT7925, Windows driver `5.7.0.6342` |
| Operating system | Windows 11 25H2, build `26200.9974`, S3 standby |
| Adapter settings | Wake on Magic Packet enabled, Wake on Pattern Match disabled, GTK Rekey for WOWLAN enabled |
| Access point | UniFi U7 Pro, firmware `8.7.11` |
| Test client | Intel 8265 on a separate computer, joining the 2.4 GHz SSID with no IPv4 address or DHCP client and IPv6 disabled |

### Newer driver and release notes

As of November 18, 2026, Microsoft Update Catalog also offers `26.40.3.69` for Windows 11 x64. Its INF is dated August 24, 2026 and lists the tested adapter's exact PCI subsystem ID. Lenovo's release notes for this version list a fix for Wi-Fi connections displaying as unavailable after a restart with Ethernet connected. They do not mention S3, LLC XID, or unexpected wakes. The sleep tests below used `5.7.0.6342`. Whether `26.40.3.69` fixes this wake remains open. [Microsoft Update Catalog package](https://www.catalog.update.microsoft.com/ScopedViewInline.aspx?updateid=ada39d67-ff35-4a47-82f0-d9e9aa59df46) · [Lenovo release notes](https://download.lenovo.com/pccbbs/mobiles/r31mw06w.html)

## What established the cause

Windows Power-Troubleshooter event 1 provided the resume time. Captures on the AP bridge and the outgoing 6 GHz interface showed where the join update traveled. An independent wired host had a magic packet scheduled for later in each trial.

| Test | What the AP observed | What Windows did |
|---|---|---|
| Ordinary client join | The same XID appeared on the bridge and the desktop's 6 GHz interface. | Woke 5.19 seconds after the frame. |
| First join during a second sleep cycle | The XID appeared on the bridge. An exact-match egress filter dropped one 20-byte frame. The 6 GHz capture saw none. | Slept through the join and disconnect. |
| Second join during that **same** sleep cycle | With the filter removed, the same client rejoined and the XID appeared on 6 GHz. | Woke 5.26 seconds after the frame, before the scheduled magic packet. |
| Independent magic-packet control | A temporary source-MAC filter dropped the test client's only broadcast on the 6 GHz interface, one 20-byte frame. The wired host later sent a magic packet. | Slept through the join and woke 4.34 seconds after the magic packet. |

A bridge firewall rule recorded zero hits and the frame still appeared on Wi-Fi. The working filter was attached to the **outgoing Wi-Fi interface**. Checking both its counter and the outgoing capture mattered here.

This identifies a trigger on one adapter, driver, and AP combination. The tests did not locate the defect inside the driver or firmware. During the reproduced wake the AP logged the joining client's WPA handshake and no group-key exchange with the sleeping desktop. A separate report describes GTK rekey wakes on the MT7922. [MT7922 GTK rekey investigation](https://github.com/Daiius/mt7922-gtk-rekey-wake)

## Recognizing the frame

On this Ethernet-format capture the destination was broadcast, the source was the joining client's MAC, the 802.3 length field was `6`, and the entire LLC payload was `00 01 af 81 01 00`. This is a normal layer-2 update that lets bridges learn where a newly associated station can be reached. The AP can emit it using the station's source address. [Linux implementation, `cfg80211_send_layer2_update`](https://linux.googlesource.com/linux/kernel/git/torvalds/linux.git/%2B/a74306e2e676f9775457366fc047a660fbf02f26/net/wireless/util.c)

For a passive check, identify the outgoing interface for the desktop's SSID and band with `iw dev`. On the tested U7 Pro it was `wifi2ap6`. Use an interface with Ethernet link type (`EN10MB`). This capture expression matches the broadcast, length, and all six LLC bytes. It leaves the joining client's MAC variable.

```sh
dev=wifi2ap6  # Substitute the interface for the sleeping PC's SSID and band.
match='ether broadcast and ether[12:2] = 6 and ether[14:4] = 0x0001af81 and ether[18:2] = 0x0100'
tcpdump -p -i "$dev" -nn -e -tttt -s 20 "$match"
```

## AP workaround

On the U7 Pro, open **UniFi Network → UniFi Devices → select the AP → Settings → Debug**, or use the AP's SSH shell. The ordinary UniFi multicast and broadcast control uses a MAC allowlist and affects other traffic, including DHCP. [Ubiquiti Debug Console instructions](https://help.ui.com/hc/en-us/articles/204909374-Connecting-to-UniFi-with-Debug-Tools-SSH) · [UniFi broadcast controls](https://help.ui.com/hc/en-us/articles/27384925962647-Managing-Broadcast-Traffic-with-UniFi)

The rule below covers updates from any joining client. It applies to **all stations on that SSID and band at this AP**. ARP, DHCP, IPv6, and ordinary magic packets have a different frame format and pass this match. A wireless bridge on that SSID would also miss these forwarding-table updates. If the desktop roams to another band or AP, traffic there follows that interface's rules.

In the same AP shell, keep `dev` and `match` set as above. Check `tc filter show dev "$dev" egress` first and use a free priority (the example uses `43137`). The rule is temporary. Recheck the interface and rule after an AP reboot or reprovision.

```sh
code=$(tcpdump -i "$dev" -ddd "$match" | tr '\n' ',' | sed 's/,$//')
tc qdisc show dev "$dev" | grep -q 'qdisc clsact' || tc qdisc add dev "$dev" clsact
tc filter add dev "$dev" egress protocol all pref 43137 handle 1 bpf bytecode "$code" flowid 1:1 action drop
tc -s filter show dev "$dev" egress
```

Check the `dropped` counter before and after a normal Wi-Fi join. This check works with the desktop awake. To verify sleep behavior, use a time when the PC is idle and arrange a known way to wake it. Check that it stays asleep across the join, then that it wakes on the intended magic packet. On Windows, `powercfg /a` reports the sleep model, and **Event Viewer → Windows Logs → System → Power-Troubleshooter → Event 1** records the resume time.

Remove only this filter with the same interface, priority, and handle.

```sh
tc filter del dev "$dev" egress protocol all pref 43137 handle 1 bpf
```

## Reporting another result

Reports from other adapters, APs, or driver versions would help establish the scope. A result on `26.40.3.69` is especially useful. Include the adapter model and driver version, Windows build and sleep model, AP model and firmware, and whether the two clients share a LAN. For each join, record whether the XID appeared on the **PC-facing** interface, whether the drop counter increased, and how long until Windows recorded a wake. Include the outcome of an intentional magic packet if you depend on WoWLAN.

For a driver comparison, keep the AP, band, sleep model, and adapter settings the same. Check the outgoing capture with the workaround removed so the new driver actually receives the frame. A quiet night with the AP filter active cannot tell which component prevented a wake. Relative timestamps and anonymized observations are enough for a public report. [Open an issue](https://github.com/Pyrynot/mt7925-unifi-wowlan-xid-wake/issues)
