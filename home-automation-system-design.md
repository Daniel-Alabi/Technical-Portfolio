# Fully Automated Home: System Design & Shopping List

*Prepared for Daniel — August 2026*

A local-first, open-source home automation system built on **Home Assistant** that ties together your existing cameras, smart lock(s), and garage opener, adds motion/presence sensing, and runs automations without monthly subscriptions or cloud dependence.

---

## 1. Architecture at a Glance

```
                        ┌─────────────────────────────┐
                        │   Home Assistant (mini PC)   │
                        │  ┌───────────┐ ┌──────────┐ │
   Internet ◄─wireguard─┤  │ Frigate   │ │ Z-Wave/  │ │
   (remote access)      │  │ NVR + AI  │ │ Zigbee   │ │
                        │  └─────┬─────┘ │ radios   │ │
                        └────────┼───────┴────┬─────┘ │
                                 │            │
        ┌──────────┬─────────────┼────────────┼──────────────┐
        │          │             │            │              │
     Cameras    Garage        Door        Motion &        Everything
     (RTSP)    (ratgdo/      lock(s)     presence         else (lights,
               blaQ)        (Z-Wave/     sensors          sirens, etc.)
                            Matter)      (Zigbee)
```

**Design principles:**

- **Local-first.** Everything critical (locks, garage, motion, camera recording) works even if your internet is down. No vendor cloud in the control path.
- **One brain.** Home Assistant is the single hub. Every device, regardless of brand, shows up in one app and one automation engine.
- **AI at the edge.** Frigate NVR does person/car/animal detection on your own hardware — your camera footage never leaves your house.
- **No subscriptions required.** Optional: Nabu Casa ($6.50/mo) for zero-config remote access, or free via Tailscale/WireGuard.

---

## 2. The Brain: Hub Hardware

| Option | ~Price | Verdict |
|---|---|---|
| **Intel N100/N150 mini PC (Beelink S12 class), 16GB RAM, 500GB NVMe** | $150–190 | ✅ **Recommended.** Handles Home Assistant + Frigate with 4–8 cameras. The N100's built-in GPU does AI object detection via OpenVINO — no extra AI accelerator needed. |
| Home Assistant Green | ~$110–120 | Great plug-and-play HA box, but too weak for camera AI. Fine if you skip Frigate. |
| Raspberry Pi 5 | ~$80–120 + accessories | Works, but SD-card wear and limited headroom for video. Only pick if you already own one. |

Install **Home Assistant OS** on the mini PC (flash and boot — about 20 minutes), then add Frigate as an add-on.

**Radios to plug into it (USB):**

- **Zigbee coordinator** — Home Assistant Connect ZBT-1 (~$35) or Sonoff Zigbee 3.0 Dongle-P (~$25). For motion sensors, door/window sensors, buttons.
- **Z-Wave stick** — Zooz ZST39 800-series (~$35). *Only needed if your lock is (or will be) Z-Wave.*

---

## 3. Cameras → Frigate NVR

Frigate turns any RTSP-capable camera into a 24/7 recorder with on-device AI detection (person, car, dog, package). How your existing cameras fit in depends on brand:

| Brand you own | Integration path | Quality |
|---|---|---|
| **Reolink, Amcrest, Hikvision, Dahua, Annke** | Native RTSP → plugs straight into Frigate + HA | ✅ Excellent, fully local |
| **Wyze** | Needs workarounds (docker-wyze-bridge) — functional but fragile | ⚠️ OK |
| **Eufy** | Community bridge (eufy-security-ws); some models do RTSP natively | ⚠️ OK |
| **Ring** | ring-mqtt add-on — live view + motion events, but cloud-dependent and no local recording | ⚠️ Limited |
| **Nest** | Google SDM API — cloud only, event-based, one-time $5 dev fee | ⚠️ Limited |

**Recommendation:** If your cameras are in the first row, you're done — zero spend. If they're Ring/Nest/Wyze, integrate what you can now, and as budget allows migrate to **Reolink PoE cameras (~$60–90 each)** which are the community favorite for Frigate. Add a **PoE switch (~$40)** if you go the wired route.

What you get: rich notifications ("Person at front door" with a snapshot), motion-triggered automations from camera zones, and a full searchable event history — all local.

---

## 4. Garage Door

Chamberlain killed the MyQ API in late 2023, so cloud control of MyQ openers via third parties is dead. The fix is a small controller wired to (or paired with) the opener itself:

| Device | Price | Fits |
|---|---|---|
| **ratgdo32** | ~$45–65 | Chamberlain/LiftMaster Security+ 1.0 & 2.0 (yellow/purple/red learn button), plus any dry-contact opener. Wires to the opener; reports open/closed/obstruction; ESPHome = fully local in HA. |
| **ratgdo32 disco** | $94 | Same, plus built-in laser parking assist & vehicle presence detection. |
| **Konnected blaQ** | $89 | Security+ 1.0/2.0 including openers with built-in myQ — pairs wirelessly like a remote, no wiring. ESPHome, fully local. |
| **Shelly 1 relay + tilt sensor** | ~$30 | Any older dry-contact opener (Genie, older Craftsman, etc.). |

⚠️ **Check your learn button color first:** Security+ 3.0 openers (white learn button, 2025 and newer) are **not supported** by ratgdo or blaQ yet.

**Recommendation:** ratgdo32 if you don't mind 10 minutes of low-voltage wiring; blaQ if you want plug-and-pair.

---

## 5. Door Lock(s)

Your existing lock likely integrates directly:

| Lock you own | Integration path |
|---|---|
| **Schlage/Yale/Kwikset Z-Wave models** | Pair with the Zooz Z-Wave stick → full local control, keypad code management from HA |
| **Schlage Encode / Encode Plus** | Matter (Encode Plus) → local via HA's Matter server; original Encode is cloud/Wi-Fi only |
| **Yale Assure with Zigbee/Z-Wave module** | Swap-in module pairs with your stick |
| **August** | HomeKit bridge trick or cloud integration — works, but cloud-dependent |
| **Kwikset Halo (Wi-Fi)** | Cloud integration only |

**If you ever replace/add a lock**, the 2026 sweet spots are the **Yale Assure Lock 2 (Z-Wave, ~$280)** for pure local control, **Schlage Encode Plus (Matter/Thread, ~$300)**, or the **Aqara U200 (~$230)** retrofit that keeps your existing keys.

What you unlock (pun intended): auto-lock at night, unlock on verified arrival, per-person keypad codes with notifications, "is everything locked?" dashboards.

---

## 6. Motion & Presence Sensors

Two kinds, used together:

- **PIR motion sensors** — cheap, battery-powered, instant. One per room/hallway.
  - **Aqara Motion Sensor P1** (~$20) — 5-year battery, adjustable sensitivity
  - **Sonoff SNZB-03P** (~$12) — budget pick
- **mmWave presence sensors** — detect *still* people (reading, sleeping, at a desk), so lights never shut off on you. Use in living room, office, bedroom.
  - **Aqara FP2** (~$60–83) — up to 30 zones per room, tracks multiple people
  - **Sonoff SNZB-06P** (~$15) — budget Zigbee presence
- **Door/window contact sensors** — **Sonoff SNZB-04P or Aqara** (~$10–15 each) for entry doors and ground-floor windows. These are the backbone of security automations.

All of these pair to the Zigbee dongle — no extra hub needed.

---

## 7. The Automations (what "fully automated" looks like)

Day one:

1. **Garage watchdog** — garage open > 10 min with nobody home → notify with camera snapshot → auto-close after 20 min (with obstruction check).
2. **Arrival** — phone GPS + car detected on driveway camera → open garage, unlock door, lights on, disarm.
3. **Departure** — last person leaves → everything locks, garage closes, cameras arm, thermostat eco.
4. **Night lockdown** — 11pm: check every lock, door, window, and the garage; anything open → notify; all clear → locks engage, exterior cameras to high-sensitivity.
5. **Person at door** — Frigate detects a person in the porch zone → snapshot notification; if you're away, play a TTS message on a speaker.
6. **Package detection** — Frigate object detection → "Package delivered" with photo.
7. **Motion lighting** — PIR turns lights on; mmWave keeps them on while the room is occupied; off 2 min after true vacancy.
8. **Security alerts** — any entry sensor opens while armed-away → siren + snapshot from nearest camera + push notification to everyone.

---

## 8. Network & Remote Access

- Put cameras on their own Wi-Fi SSID or VLAN with **no internet access** — Frigate talks to them locally.
- Remote access: **Nabu Casa** ($6.50/mo, supports HA development, zero config) or free **Tailscale/WireGuard** VPN.
- Enable HA's built-in backups to a USB drive or cloud — set and forget.

---

## 9. Shopping List

### Core (required)

| Item | Price |
|---|---|
| Intel N100/N150 mini PC, 16GB/500GB | $150–190 |
| Zigbee coordinator (ZBT-1 or Sonoff Dongle-P) | $25–35 |
| Garage controller (ratgdo32 or Konnected blaQ) | $45–94 |
| **Core total** | **~$220–320** |

### Sensors (scale to your home — prices per unit)

| Item | Price |
|---|---|
| Aqara P1 motion sensor ×4–6 | $20 ea |
| Aqara FP2 mmWave presence ×1–2 (main living areas) | $60–83 ea |
| Sonoff/Aqara door & window contact sensors ×6–10 | $10–15 ea |
| **Typical sensor spend** | **~$220–350** |

### Conditional (depends on your existing gear)

| Item | When | Price |
|---|---|---|
| Zooz ZST39 Z-Wave stick | Your lock is Z-Wave | $35 |
| Reolink PoE camera | Replacing cloud-locked cameras | $60–90 ea |
| 8-port PoE switch | Going wired cameras | ~$40 |
| Shelly 1 + tilt sensor | Older dry-contact garage opener | ~$30 |
| Zigbee siren (e.g., Heiman) | Audible alarm | ~$25 |

**Realistic all-in range: ~$450–700** using your existing cameras and lock, with no recurring fees.

---

## 10. Build Roadmap

**Weekend 1 — The brain.** Set up the mini PC with Home Assistant OS, plug in the Zigbee dongle, install the companion app on your phones, pair 2–3 sensors, make your first motion-light automation.

**Weekend 2 — Cameras.** Install Frigate, connect your cameras (or order replacements if they're cloud-locked), configure person detection zones, set up snapshot notifications.

**Weekend 3 — Garage & locks.** Install the ratgdo/blaQ, pair the lock (Z-Wave stick if needed), build the arrival/departure and night-lockdown automations.

**Weekend 4 — Polish.** Dashboards for wall tablets/phones, presence sensors in main rooms, VPN remote access, backups, and the security/alarm mode (HA's Alarmo add-on is excellent for this).

---

## Notes & Caveats

- Verify your garage opener's **learn button color** before ordering (white = Security+ 3.0 = wait, not yet supported).
- Tell me your camera and lock **brands/models** and I'll give you the exact integration steps for each.
- Prices are August 2026 street prices (US) and fluctuate; Aqara gear frequently dips 20–30% in sales.
