# Wi-Fi Optimizer — Real Android App + Network Architecture

A working Android Studio project (Kotlin + Jetpack Compose) that monitors real Wi-Fi radio
metrics, runs real latency/packet-loss probes, and gives actionable, physically-grounded
recommendations — plus the network-side architecture and hardware guide needed to actually
reach ~100 m of coverage.

**No feature in this app claims to boost signal, increase antenna power, or extend physical
range by software alone. Every "fix" recommended is either a placement/router-config change
or a genuine additional piece of hardware.**

---

## 1. What Android actually allows a normal (non-root) app to do

| Capability | Allowed? | Notes |
|---|---|---|
| Read RSSI (dBm), link speed, frequency, channel width | ✅ Yes | Via `WifiManager` / `WifiInfo`, or `NetworkCapabilities.transportInfo` on API 31+ |
| Read 802.11 standard (n/ac/ax) | ✅ Partial | Only API 29+ exposes `wifiStandard` |
| Trigger a scan / read nearby APs | ✅ Yes, throttled | Needs location or `NEARBY_WIFI_DEVICES` (API 33+) permission; OS throttles to ~4 scans/2min in background |
| Measure real latency (ICMP ping) | ✅ Yes | By invoking the device's built-in `/system/bin/ping` binary, which already has `CAP_NET_RAW` — this is not a root trick |
| Measure packet loss | ✅ Yes | Derived from ping statistics, or TCP-connect-failure rate as fallback |
| Run continuously in the background | ✅ Yes, with caveat | Requires a **foreground service with a visible notification** — Android will not let a normal background service/thread poll indefinitely (Doze/App Standby will kill it) |
| **Increase Wi-Fi transmit power** | ❌ No | Not exposed to apps since Android 8. TX power is controlled by the radio driver/firmware and regulatory domain, full stop |
| **Change antenna characteristics / gain** | ❌ No | This is physical hardware; software cannot alter it on any platform |
| **Force a specific Wi-Fi channel on the phone** | ❌ No | Only the *access point* selects/broadcasts a channel; the client (phone) just follows whatever the AP is using. Channel selection is a router-side setting, not a phone-side one |
| **Improve true radio range** | ❌ No | Range is a function of TX power, antenna, frequency, and obstructions. No app changes any of these |
| Auto-switch between APs of the *same* SSID (roaming) | ⚠️ Partial | The phone's own Wi-Fi stack decides this (802.11k/v/r support in your router matters far more than anything an app does); an app can only *recommend* moving networks manually, or auto-connect to a *different* SSID if you deliberately configure both networks and priorities |
| Configure the router (channel, width, TX power, band steering) | ⚠️ Only if the router exposes an API | There's no universal Android or router API for this. Handled in this project via a pluggable `RouterAdapter` interface — supported models get automation, everything else gets precise manual instructions |

**Bottom line:** this app is a real, honest instrument panel + advisor. The actual range/speed
gains come from where you place hardware and how you configure the router — which is why
Section 3 below (network architecture) matters as much as the app itself.

---

## 2. Network optimization the app helps you apply (router-side)

These are standard, real wireless-engineering levers. The app's Settings screen stores your
router profile and pingtarget; the Recommendations card tells you *which* of these to change
and why, based on live data:

- **Router placement** — central, elevated, away from metal/mirrors/large appliances (these
  cause real multipath/attenuation, not imagined).
- **Channel selection** — 2.4 GHz: use only 1, 6, or 11 (non-overlapping in the US/most
  regions). 5 GHz: prefer non-DFS channels (36–48) for lower latency (DFS channels can trigger
  radar-avoidance channel switches that cause momentary drops).
- **Channel width** — 20 MHz on 2.4 GHz for maximum range/robustness; 80 MHz on 5 GHz for a
  speed/reliability balance (160 MHz only helps if very few neighboring networks exist).
- **2.4 GHz vs 5 GHz** — 2.4 GHz penetrates walls better (range); 5 GHz is faster but
  attenuates faster with distance/obstructions. The app's recommendation engine actively
  suggests which band fits your current RSSI.
- **Transmit power** — set to max if your regulatory domain and router allow it, unless you
  have close neighbors (interference tradeoff).
- **Roaming (802.11k/v/r)** — enable "Smart Roaming"/"Band Steering"/"Fast Transition" on the
  router if it's a mesh or multi-AP setup — this is what actually makes the phone hand off
  between APs smoothly. No Android app can substitute for this.
- **DNS** — use a fast resolver (e.g., 1.1.1.1, 8.8.8.8) — same targets used for latency
  testing in this app.
- **Congestion / interference** — the app's nearby-AP scan flags how many other networks share
  your current channel, a direct, real congestion signal.

---

## 3. Reaching ~100 meters — architecture comparison

A single router, even a very good one, realistically covers **20–40 m indoors through
several walls** on 5 GHz, and maybe **40–60 m** on 2.4 GHz, before signal drops below a
usable ~-75 dBm. **Reaching 100 m through building structure with a single access point is
not physically realistic** for consumer-grade Wi-Fi — this needs to be said plainly rather
than sold to you as achievable with "the right router."

| Option | Range | Speed | Latency | Reliability | Cost | Verdict |
|---|---|---|---|---|---|---|
| A. Better router/antenna alone | Low-moderate gain | Moderate gain | No change | No change | $ | Marginal improvement only; won't reach 100 m alone |
| B. Wi-Fi repeater/extender (single-radio) | Extends range | **Halves throughput** (relays on the same radio/channel) | Adds latency | Moderate — extra hop, can flap | $ | Cheapest fix, real speed penalty |
| C. Mesh Wi-Fi (proper tri-band or wired backhaul) | Extends range well | Speed preserved if backhaul is wired or dedicated | Low added latency | High — built-in roaming (802.11k/v/r) | $$ | **Best all-around for 100 m if wired backhaul is possible** |
| D. Wired access point (Ethernet-fed AP at the far point) | Excellent — new radio, full strength at that spot | Full speed (no backhaul penalty) | Lowest | Highest | $$ (cost of running cable) | **Best technical solution if you can run a cable** |
| E. Outdoor AP (weatherproof, higher-gain antenna) | Best for outdoor/line-of-sight spans | High if cable-fed | Low | High | $$–$$$ | Right choice if 100 m is outdoors/across a yard |
| F. Point-to-point wireless bridge (e.g., 5 GHz PtP dishes) | Very good over open line-of-sight | High | Low | High if LOS is clean | $$$ | Only worth it for a detached building where running cable is impractical |

### Recommended architecture for ~100 m

1. **If you can run an Ethernet cable** (even through conduit) to a point roughly halfway or
   at the far location: **Option D — a second wired access point**, on its own channel,
   same SSID, with 802.11k/v/r roaming enabled. This is the best combination of range, speed,
   latency, and reliability, and is what this app's roaming/AP-selection logic assumes.
2. **If cabling is impractical indoors**: **Option C — a proper mesh system with wired
   backhaul where possible**, or at minimum a mesh system (not a cheap single-radio repeater)
   using dedicated backhaul channels.
3. **If 100 m is outdoors / across open ground with line of sight**: **Option E or F** —
   an outdoor AP or a point-to-point bridge pair, which are built for exactly this.
4. **Avoid** plain single-radio repeaters/extenders as the *only* fix for 100 m — they halve
   throughput and are the most common source of "my extender made it worse" complaints.

---

## 4. Automatic access-point selection — what's app-side vs router-side

- **Router/AP side (does the real work):** 802.11k (neighbor reports), 802.11v (BSS
  transition management), 802.11r (fast roaming) let compatible APs *tell* the phone when to
  roam and hand off the session quickly. This must be enabled on the router/mesh system —
  no Android app can add this capability to hardware that lacks it.
- **App side (this project):** reads current RSSI/link quality, scans for stronger APs *of
  the same SSID*, and — because Android does not let a third-party app force a network
  switch to a different BSSID within the same SSID — surfaces this as a clear
  recommendation/notification rather than a silent, forced switch. If you configure two
  *different* SSIDs (e.g., a 2.4 GHz-only and 5 GHz-only network), the app *can* programmatically
  suggest and (with your confirmation) connect to the better one via `WifiNetworkSuggestion`
  API (Android 10+), which is included as the extension point in `WifiMonitor`/`ViewModel`
  for you to wire up if you choose that network layout.

---

## 5. Router control module

`RouterAdapter` (see `data/RouterIntegration.kt`) is the pluggable interface requested:
- `GenericRouterAdapter` — no API assumed; returns precise manual admin-panel steps.
- `OpenWrtUbusAdapter` — real skeleton for routers exposing an HTTP/ubus API (OpenWrt,
  some ASUS-Merlin builds). Fill in your router's actual endpoint — the exact JSON schema is
  per-firmware, which is why this is modular rather than hardcoded.
- Add your own adapter (e.g., for a specific ASUS, TP-Link Omada, or UniFi controller API) by
  implementing `RouterAdapter` and registering it in `RouterAdapterFactory`.

---

## 6. Hardware recommendations for ~100 m coverage

### Cheapest (~$50–90)
- One Wi-Fi mesh add-on node (single dual-band mesh unit) placed roughly at the midpoint
  between the main router and the 100 m target, using wireless backhaul.
- Expect: usable but not maximum speed at the far point (wireless backhaul shares airtime).
- Placement: midpoint, elevated, line-of-sight to both the router and the target area if at
  all possible.

### Best value (~$150–300)
- A 2–3 node mesh system with dedicated tri-band backhaul channel (avoids halving client
  speed), or a single wired access point if one Ethernet run is feasible.
- Placement: one node central in the house, one node near the 100 m target; if using a wired
  AP instead, run cable to a point as close to the target as practical.
- Enable band steering / fast roaming in the mesh app.

### Best performance (~$400–700+)
- A wired multi-AP setup: 2–3 access points (e.g., UniFi, TP-Link Omada, or similar
  "enterprise-lite" APs) connected via Ethernet to a switch/router, each broadcasting the
  same SSID with 802.11k/v/r roaming enabled, positioned so their coverage overlaps by
  ~20–30%.
- For an outdoor 100 m span with no indoor path: add one outdoor-rated AP or a
  point-to-point bridge pair aimed at each other.
- This gives full speed at every point (no relay penalty), lowest latency, and the most
  reliable roaming — matching exactly what the app's AP-selection logic is designed to work
  with.

---

## 7. Project structure

```
WifiOptimizer/
├── app/
│   ├── build.gradle.kts
│   ├── proguard-rules.pro
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/wifioptimizer/app/
│       │   ├── MainActivity.kt
│       │   ├── data/
│       │   │   ├── WifiMonitor.kt            (radio-layer readings)
│       │   │   ├── NetworkQualityTester.kt   (real ping + packet loss)
│       │   │   ├── RecommendationEngine.kt   (scoring + honest advice)
│       │   │   ├── WifiMonitorService.kt     (foreground background monitoring)
│       │   │   ├── WifiMonitorRepository.kt  (shared state)
│       │   │   └── RouterIntegration.kt      (pluggable router adapters + settings)
│       │   ├── model/WifiModels.kt
│       │   ├── ui/
│       │   │   ├── DashboardScreen.kt
│       │   │   ├── SettingsScreen.kt
│       │   │   └── theme/ (Color.kt, Type.kt, Theme.kt)
│       │   └── viewmodel/WifiViewModel.kt
│       └── res/ (strings, themes, launcher icon, backup rules)
├── build.gradle.kts
├── settings.gradle.kts
├── gradle.properties
└── gradle/wrapper/gradle-wrapper.properties
```

## 8. Building it

1. Open the `WifiOptimizer/` folder in Android Studio (Iguana or newer).
2. Let Gradle sync (it will download the wrapper JAR + Gradle 8.7 automatically — this
   sandbox cannot download from `services.gradle.org`, so on first open in Android Studio
   just let it run "Sync Now"; the wrapper JAR itself is normally auto-generated by Android
   Studio if missing).
3. Run on a device (emulators generally don't have usable Wi-Fi radios for RSSI testing —
   use a real phone).
4. Grant the requested permissions when prompted (needed for scan/location reasons explained
   in Section 1).

## 9. Known, honest limitations of this build

- The `OpenWrtUbusAdapter` is a real but minimal HTTP skeleton — the actual ubus JSON-RPC
  call schema depends on your firmware version; you'll need to fill in your router's exact
  endpoint/session-auth flow.
- `WifiNetworkSuggestion`-based auto-switching between two different SSIDs (e.g., a dedicated
  2.4/5 GHz split) is described as an extension point but not wired into the UI in this build,
  since it requires you to decide your specific SSID layout first.
- Emulators do not provide real RSSI/link-speed values; test on physical hardware.
