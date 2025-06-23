# Network
Linux, SSH, Proxy, Burp

# 🔐 Burp Suite + Firefox Proxy + SOCKS5 Tunnel via AWS (No IP Leaks)

## 🎯 Cel

Przechwycić cały ruch HTTP/HTTPS z przeglądarki Firefox w **Burp Suite**, przy jednoczesnym jego tunelowaniu przez **SOCKS5** (np. zdalny serwer AWS), tak aby:

- Burp przechwytywał cały ruch
- Publiczny IP widoczny z przeglądarki był tunelowany
- DNS oraz WebRTC nie ujawniały lokalnego IP
- System i inne aplikacje działały niezależnie

---

## 📦 Wymagania

- Burp Suite Community / Pro
- Firefox + FoxyProxy
- Privoxy
- Serwer zdalny (np. AWS EC2 z otwartym portem SSH)
- System Linux (testowane na Ubuntu/Debian)

---

## 🧱 Architektura

```
Firefox (HTTP Proxy: 127.0.0.1:8118)
    ↓
Privoxy (forward to 127.0.0.1:8080)
    ↓
Burp Suite (SOCKS5: 127.0.0.2:1080)
    ↓
SSH Tunnel → Internet (np. AWS )
```

---

## 🧰 Konfiguracja krok po kroku

### 1️⃣ SSH Tunnel (SOCKS5)

```bash
sudo ip addr add 127.0.0.2/8 dev lo
ssh -D 127.0.0.2:1080 user@aws-host -N
```

- Dodaje alias IP na localhost (127.0.0.2), żeby nie kolidować z innymi usługami.
- Tworzy SOCKS5 proxy przez SSH na zdalnym serwerze (np. AWS EC2).

---

### 2️⃣ Burp Suite

- Otwórz Burp Suite.
- Przejdź do **Proxy → Options → Proxy Listeners**.
- Ustaw listener na `127.0.0.1:8080` (domyślnie).
- (Opcjonalnie) Wyłącz inne listenery.

---

### 3️⃣ Privoxy

#### Instalacja

```bash
sudo apt update
sudo apt install privoxy
```

#### Konfiguracja `/etc/privoxy/config`

Dodaj (lub zamień) linie:
```
listen-address  127.0.0.1:8118
forward-socks5   / 127.0.0.2:1080 .
forward          / 127.0.0.1:8080 .
```
**Wyjaśnienie:**  
- Privoxy nasłuchuje na 127.0.0.1:8118.
- Forwarduje ruch HTTP do Burpa (127.0.0.1:8080), a SOCKS przez 127.0.0.2:1080.

#### Restart Privoxy

```bash
sudo systemctl restart privoxy
```

---

### 4️⃣ Firefox + FoxyProxy

1. Zainstaluj [FoxyProxy](https://addons.mozilla.org/pl/firefox/addon/foxyproxy-standard/).
2. Ustaw nowy profil proxy:
   - Typ: HTTP
   - Host: 127.0.0.1
   - Port: 8118
3. Ustaw FoxyProxy na ten profil.
4. W about:config ustaw:
   - `media.peerconnection.enabled` → **false** (blokuje WebRTC leaks)
   - `network.proxy.socks_remote_dns` → **true** (DNS przez SOCKS5)
   - (Opcjonalnie) `network.proxy.allow_hijacking_localhost` → **true**

---

## 🧪 Testowanie

- Wejdź na [https://ifconfig.me](https://ifconfig.me) — powinna pojawić się IP z AWS.
- W Burp Suite powinieneś widzieć cały ruch z przeglądarki.
- Sprawdź [https://browserleaks.com/webrtc](https://browserleaks.com/webrtc) — lokalny IP nie powinien być widoczny.

---

## 🚑 Problemy

- **Brak ruchu w Burp:** Sprawdź kolejność forwardów w Privoxy i ustawienia listenera Burpa.
- **Nie działa SOCKS:** Upewnij się, że tunel SSH działa i nasłuchuje na 127.0.0.2:1080.
- **Wycieki DNS:** Sprawdź ustawienia `network.proxy.socks_remote_dns`.
- **WebRTC Leaks:** Upewnij się, że `media.peerconnection.enabled` jest wyłączone.

---

## 🧹 Sprzątanie

Po zakończeniu:
```bash
sudo ip addr del 127.0.0.2/8 dev lo
```
Wyłącz SSH tunel oraz Privoxy.

---

## 💡 Notatki

- Jeśli chcesz całość uruchomić tylko dla wybranego profilu Firefox, trzymaj osobny profil.
- Można też użyć innych narzędzi do tunelowania SOCKS5.
- Privoxy pozwala na łatwą inspekcję i modyfikację ruchu przed Burpem.

---
