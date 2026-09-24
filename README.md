# 🎧 Scream Audio Monitoring

📄 **Project page:** [Integrated Audio Monitoring System — Scream Ecosystem](https://zirize.github.io/scream-audio-monitoring/) · More projects: [zirize.github.io](https://zirize.github.io/)

This guide shows how to collect audio alerts and system sounds from many devices (Windows, Linux, embedded) on a LAN and play them at one monitoring station, with no audio cables. It uses the **Scream protocol**, which carries uncompressed PCM audio in UDP packets (multicast or unicast).

Typical uses:

- Hearing alarms from many PCs or embedded devices in one place (smart factories, server rooms, labs)
- Getting audio out of a VM onto the host without passthrough hardware
- Playing PC audio on an Android phone over Wi‑Fi

---

## 🧩 Components

| Role | Project | Platform | Notes |
|------|---------|----------|-------|
| Sender | [duncanthrax/scream](https://github.com/duncanthrax/scream) | Windows | Virtual sound card driver. Also includes ScreamReader and the Unix receiver |
| Sender | [zirize/pipewire-scream](https://github.com/zirize/pipewire-scream) | Linux (PipeWire) | Virtual sink that streams whatever plays into it |
| Sender | [zirize/screamplay](https://github.com/zirize/screamplay) | Linux CLI | Plays audio files (WAV/FLAC/OGG/AIFF/MP3) onto the network and resamples them |
| Receiver | [zirize/screamdroid](https://github.com/zirize/screamdroid) | Android | Foreground service, mutes automatically during calls, buffer adapts to the network |
| Receiver | `Receivers/unix` in duncanthrax/scream | Linux | Plays through PulseAudio / ALSA / JACK |

## 🗺️ Architecture

```text
 ┌──────────────┐   ┌──────────────────┐   ┌──────────────┐
 │ Windows PC   │   │ Linux PC         │   │ Linux script │
 │ Scream driver│   │ pipewire-scream  │   │ screamplay   │
 └──────┬───────┘   └────────┬─────────┘   └──────┬───────┘
        │   UDP multicast 239.255.77.77:4010      │
        └──────────────┬─────┴────────────────────┘
                       ▼
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
  ScreamDroid     Unix receiver     ScreamReader
   (Android)        (Linux)          (Windows)
```

> ⚠️ Two or more senders on the same multicast group **collide**, and the audio breaks up. To monitor several sources at once, use the **central relay** setup below.

## ⚡ Quick Start

1. **Pick a sender**
   - Windows: install the Scream driver, then set the registry values. On Windows 10/11 this step is required ([details](https://zirize.github.io/scream-audio-monitoring/#component-1))
   - Linux: build and install `pipewire-scream`, then route audio to the new sink
   - Script alerts: `./screamplay alert.wav`
2. **Pick a receiver**
   - Android: install the APK from [ScreamDroid Releases](https://github.com/zirize/screamdroid/releases)
   - Linux: build `Receivers/unix`, then run `scream` (add `-i <iface>` to choose an interface)
3. **Check that packets arrive** (on the receiver host)
   ```bash
   sudo tcpdump -n -i any udp port 4010
   ```
   If packets are arriving, `length 1157` lines should scroll by.
4. **Open the firewall** (if nothing arrives)
   ```bash
   sudo ufw allow 4010/udp                     # Ubuntu/Debian
   sudo firewall-cmd --add-port=4010/udp        # Fedora/RHEL (add --permanent to keep it)
   ```

## 🎛️ Central relay (mixing multiple sources)

```text
 PC1 ──unicast:4011──┐
 PC2 ──unicast:4012──┤→ Linux relay: Unix receiver × N → shared sink (mix)
 VM  ──unicast:4013──┘        → pipewire-scream → multicast:4010 → receivers
```

- Each sender sends **unicast** to its own port on the relay (Windows: `UnicastIPv4` and `UnicastPort` registry values)
- The relay runs one Unix receiver per port (e.g. `scream -u -p 4011`) and mixes them into one sink
- `pipewire-scream` re-sends the mix as a single multicast stream. Receivers such as ScreamDroid then have only one stream to listen to

Full walkthrough: [pipewire-scream relay setup guide](https://zirize.github.io/pipewire-scream/#advanced-audio-relay-mixing)

## 📡 Protocol summary

| Item | Value |
|------|-------|
| Transport | UDP, multicast by default (unicast optional) |
| Default address | `239.255.77.77:4010` |
| Packet | 5-byte header + 1152-byte PCM = **1157 bytes** |
| Header | `[0]` sample rate (bit7: 0=48 kHz base, 1=44.1 kHz base; bits 0–6: multiplier) · `[1]` bit depth · `[2]` channels · `[3–4]` channel mask |
| Bandwidth | 48 kHz/16-bit stereo ≈ 1.5 Mbps · 5.1ch ≈ 4.6 Mbps |

## 🩺 Troubleshooting

| Symptom | Check |
|---------|-------|
| No sound at all | Run `tcpdump` to see whether packets arrive → open UDP 4010 in the firewall → check that the router/switch allows multicast (IGMP). Try unicast mode if needed |
| Windows sends nothing | Check the `HKLM\SYSTEM\CurrentControlSet\Services\Scream\Options` registry values and **reboot** |
| Broken or garbled sound | Make sure only one sender is using the same group/port. For multiple senders, use the relay setup |
| Stuttering / dropouts | Use a wired connection, check Wi‑Fi multicast quality, or raise the receiver's buffer |
| Multichannel is unstable | Use 48 kHz / 16-bit for 5.1 and above. Avoid 24/32-bit |

## 📂 Repository layout

```text
.
├── index.html               # Project page (GitHub Pages)
├── scream_infographic.png   # Architecture infographic
├── scream_mindmap.png       # Protocol mind map
├── sitemap.xml, robots.txt  # SEO
└── README.md                # This file
```

## 🙏 Credits

- Scream protocol and Windows driver: [Tom Kistner (duncanthrax)](https://github.com/duncanthrax/scream)
- The infographic and mind map were made with NotebookLM
