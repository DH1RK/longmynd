# HowTo: Longmynd-Server auf Raspberry Pi 4B (64-Bit)

Installationsanleitung für Longmynd mit Web-UI und WebSocket-Steuerung
(Longmynd Client von ZR6TG, OpenTuner) auf einem Raspberry Pi 4B mit
MiniTiouner (Express). Quelle ist der Fork `DH1RK/longmynd`. Er basiert auf
`philcrump/longmynd` und enthält zusätzlich den Fix aus dem Pull Request #2
von dl1rf. Es ist kein weiterer Patch nötig (Hintergrund in Abschnitt 5).

Stand: November 2026. Getestet mit Raspberry Pi OS (64-bit), GCC 14.2.0.

## Überblick

| Komponente | Wert in diesem Setup |
|---|---|
| Hardware | Raspberry Pi 4B (4 GB), MiniTiouner Express an USB 3.0 (blau) |
| Betriebssystem | Raspberry Pi OS (64-bit), Lite genügt (headless) |
| Hostname / IP | `longmynd-server` / `<RPi-IP>` (z. B. `192.168.2.158`) |
| Benutzer | `dh1rk` |
| Longmynd-Quelle | `https://github.com/DH1RK/longmynd.git` |
| Enthaltener Fix | PR #2 von dl1rf (`json_object_new_double` → `json_object_new_int`), bereits im Fork |
| Steuerung + Status | WebSocket, Port 8080 |
| Video (UDP) | wird vom Client gesetzt, hier Port 4003 an den PC |

> **Warum 64-Bit?** Der Pi 4B ist ein 64-Bit-Gerät. „Legacy" im Imager
> bedeutet nur ein älteres Debian-Release. Longmynd ist reines C und läuft
> auf 32 und 64 Bit.

## 1. SD-Karte erstellen

Das Erstellen der SD-Karte mit dem Raspberry Pi Imager (v2.0.12) ist in
[01_RaspberryPi_Imager.md](01_RaspberryPi_Imager.md) beschrieben. Kurzfassung:
Gerät **Raspberry Pi 4**, Betriebssystem **Raspberry Pi OS (64-bit)** (Lite
genügt), in der Anpassung **Hostname**, **Benutzername und Passwort**,
**WLAN** (falls kein LAN-Kabel) setzen und **SSH aktivieren**. Danach
schreiben, SD-Karte in den Pi stecken und einschalten.

Ein Ethernet-Kabel ist stabiler als WLAN. Bei Problemen mit dem ersten Login
zuerst Kabel testen (häufigster Fehler: Tippfehler bei SSID/Passwort).

## 2. Pi im Netzwerk finden und anmelden

> **Beispielwerte:** Die Anmeldung in diesem Abschnitt zeigt das Setup des
> Autors als Beispiel: Hostname `longmynd-server`, Benutzer `dh1rk`, IP
> `192.168.2.158`. Ersetzen Sie sie durch Ihre eigenen Werte aus dem Imager
> (Hostname und Benutzername) bzw. aus Ihrem Netzwerk (`<RPi-IP>`).

Windows (PowerShell):

```powershell
ping longmynd-server.local          # Hostname aus dem Imager
ssh dh1rk@longmynd-server.local
```

`longmynd-server` ist der **Hostname** aus dem Imager (Standard wäre
`raspberrypi`), nicht der Benutzername und nicht die Software. Falls der
Name nicht aufgelöst wird: IP im Router nachsehen (Fritz!Box:
Heimnetz → Netzwerk) oder `arp -a` prüfen und mit `ssh dh1rk@<IP>` verbinden.

### Anmeldung per SSH (Logon)

Das Format ist immer `ssh <benutzer>@<hostname-oder-IP>`. Benutzername und
Passwort sind die Werte, die du im Imager gesetzt hast.

```powershell
ssh <benutzer>@<hostname-oder-IP>
# Beispiel mit Hostname:  ssh dh1rk@longmynd-server.local
# Beispiel mit IP:        ssh dh1rk@192.168.2.158
```

Ablauf beim ersten Login:

1. Die Frage nach dem Host-Key (`Are you sure you want to continue connecting
   (yes/no/[fingerprint])?`) mit `yes` bestätigen. Der Fingerprint wird in
   `C:\Users\<Name>\.ssh\known_hosts` gespeichert.
2. Das Passwort eingeben. Dabei erscheinen **keine Zeichen** und keine Sternchen.
   Das ist normal. Mit Enter bestätigen.
3. Es erscheint der Prompt `<benutzer>@<hostname>:~ $` (Beispiel: `dh1rk@longmynd-server:~ $`). Ab hier laufen alle
   Befehle der folgenden Schritte **auf dem Pi**.

Abmelden mit `exit` oder `Strg+D`. Windows 11 bringt den OpenSSH-Client
bereits mit (`ssh` und `scp` in PowerShell). Alternativ geht PuTTY.

Wenn die Anmeldung scheitert:

| Meldung / Symptom | Ursache und Abhilfe |
|---|---|
| `Could not resolve hostname` | `.local`-Name wird nicht aufgelöst. Mit der IP verbinden. |
| `Connection timed out` / `refused` | Pi noch nicht gebootet, falsche IP oder SSH im Imager nicht aktiviert. Kabel und LED prüfen. |
| `Permission denied, please try again` | Benutzername oder Passwort falsch. Der Benutzername ist der aus dem Imager, nicht der Hostname. |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | SD-Karte neu geflasht, der Host-Key ist neu. Alten Eintrag löschen: `ssh-keygen -R longmynd-server.local` und ggf. `ssh-keygen -R <RPi-IP>`. |

Optional kann die IP im Router fest vergeben werden (DHCP-Reservierung), damit
sie sich nicht ändert.

### Wo läuft welcher Befehl?

Ab hier arbeitest du mit zwei Fenstern. Woran du erkennst, wo du bist:

| Fenster | Prompt | Zweck |
|---|---|---|
| **Windows-PowerShell** | `PS D:\...>` | `ssh`, `scp`, Browser, Wireshark, `Get-NetUDPEndpoint` |
| **SSH-Sitzung zum Pi** | `<benutzer>@<hostname>:~ $` (z. B. `dh1rk@longmynd-server:~ $`) | alle Befehle der Schritte 3–10 (`apt`, `git`, `make`, `systemctl`) |

Alle Befehle in dieser Anleitung, die mit `sudo`, `apt`, `git`, `make`,
`systemctl`, `journalctl` oder `./longmynd` beginnen, laufen **auf dem Pi**.
Befehle mit `ssh`, `scp`, `ping`, `Test-NetConnection` und `Get-...` laufen
**auf dem Windows-PC**.

### Dateien zwischen Pi und Windows kopieren (scp)

`scp` ist Teil von OpenSSH und kopiert über die SSH-Verbindung. Starte es
immer **von Windows aus** in einer PowerShell, nicht im SSH-Fenster. Der Pi
hat einen SSH-Server, Windows standardmäßig nicht. Läuft `scp` auf dem Pi und
zeigt auf einen Windows-Pfad, scheitert es mit
`Could not resolve hostname d: Name or service not known`.

**Wichtig:** Im Zielpfad darf kein Laufwerksbuchstabe mit Doppelpunkt stehen
(`D:\...`). `scp` liest alles vor dem `:` als Rechnernamen. Wechsle deshalb
zuerst in den Zielordner und verwende `.` als Ziel.

Datei **vom Pi nach Windows** holen:

```powershell
cd <zielordner>
scp <benutzer>@<hostname-oder-IP>:~/pi_history.txt .
# Beispiel:  cd D:\repos\longmynd\docs
#            scp dh1rk@longmynd-server.local:~/pi_history.txt .
```

Datei **von Windows zum Pi** schicken (Quelle relativ, Ziel nach dem `:` ist
der Pfad auf dem Pi):

```powershell
cd <quellordner>
scp <datei> <benutzer>@<hostname-oder-IP>:~/
# Beispiel:  cd D:\repos\longmynd\docs
#            scp HowTo_Install.md dh1rk@longmynd-server.local:~/
```

Ganzen Ordner kopieren: Option `-r`, zum Beispiel
`scp -r <benutzer>@<hostname-oder-IP>:~/longmynd/web/htdocs .` (Beispiel: `scp -r dh1rk@longmynd-server.local:~/longmynd/web/htdocs .`)

Befehlshistorie des Pi exportieren: zuerst im **SSH-Fenster**
`history > ~/pi_history.txt`, dann in der **PowerShell** das `scp` von oben.
Vor dem Weitergeben die Datei durchsehen, denn sie enthält jeden getippten
Befehl, auch Passwörter, falls welche in einem Befehl standen.

## 3. System aktualisieren, Abhängigkeiten installieren

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git make gcc cmake libusb-1.0-0-dev libasound2-dev libjson-c-dev
```

`cmake` wird gebraucht, weil der Makefile libwebsockets aus dem Submodul
selbst baut. Nach einem Kernel-Update `sudo reboot` und neu anmelden.

## 4. Quellcode holen

```bash
cd ~
git clone --recursive https://github.com/DH1RK/longmynd.git
cd longmynd
```

`--recursive` ist Pflicht, sonst fehlt libwebsockets (Submodul unter
`web/libwebsockets`).

## 5. Hintergrund: Unterschied zu philcrump und BATC

Dieser Abschnitt ist nur Hintergrund. Für die Installation ist nichts zu tun,
der Fork `DH1RK/longmynd` ist direkt baubar.

### 5.1 Herkunft des Codes

```
myorangedragon/longmynd   (BATC-Original)
├── F5OEO/longmynd        (Variante des Originals, kein Web-UI)
└── philcrump/longmynd    (Weiterentwicklung mit Web-UI und WebSocket)
    ├── dl1rf/longmynd    (Fork mit dem Fix aus PR #2)
    └── DH1RK/longmynd    (dieser Fork: philcrump + PR #2 von dl1rf)
```

- **BATC-Original** und die `F5OEO`-Variante haben kein Web-UI und keine
  WebSocket-Steuerung und taugen für diesen Zweck nicht.
- **`philcrump/longmynd`** bringt Web-UI und WebSocket (Port 8080) mit.
- **`dl1rf/longmynd`** ist Phils Stand plus ein Commit (Ganzzahlen im JSON).
- **`DH1RK/longmynd`** ist Phils Stand plus der Fix von dl1rf.

### 5.2 Der Fix aus PR #2 (`web/web.c`)

`philcrump/longmynd` sendet im WebSocket-JSON alle Zahlen als `double`
(`7.0`). Der Longmynd Client und OpenTuner erwarten Ganzzahlen und zeigen
dann keinen brauchbaren Status. PR #2
(https://github.com/philcrump/longmynd/pull/2, Autor dl1rf) stellt
`web/web.c` auf `json_object_new_int` um. Stand dieser Anleitung: der PR ist
bei Phil **offen, nicht gemergt**. Im Fork `DH1RK/longmynd` ist er per Merge
enthalten (der Commit von dl1rf bleibt mit seinem Namen im Verlauf).

Was sich dadurch ändert: Alle Statuswerte (Frequenz, Symbolrate, MER, BER,
PIDs, Konstellation, Ports) werden als `int` ausgegeben. Der Zeitstempel
wird auf ganze Sekunden gekürzt. Die Quellwerte sind ohnehin ganzzahlig.

Prüfen, ob der Fix im Klon steckt:

```bash
grep -c json_object_new_int web/web.c     # Ergebnis größer als 0
```

### 5.3 Makefile: keine Änderung nötig

Der Makefile des Forks `DH1RK/longmynd` (unverändert von `philcrump/longmynd`) verwendet `-O3 -march=native
-mtune=native` und enthält **keine** 32-Bit-spezifischen Optionen. Er baut
unverändert auf aarch64.

> Nur falls du je einen Makefile mit den Zeilen
> `COPT_RPI2 = -mfpu=neon-vfpv4` / `COPT_RPI34 = -mfpu=neon-fp-armv8`
> findest (das stammt aus dem BATC-Original): Auf 64-Bit schlägt der Build
> mit `unrecognized command-line option '-mfpu=...'` fehl. Abhilfe:
> ```bash
> sed -i 's/^COPT_RPI2 = .*/COPT_RPI2 =/; s/^COPT_RPI34 = .*/COPT_RPI34 =/' Makefile
> ```
> Für diesen Fork wird das **nicht** benötigt.

## 6. Kompilieren

```bash
make
ls -l longmynd
```

Beim ersten Mal wird zuerst libwebsockets gebaut (einige Minuten, viele
Zeilen Ausgabe). Am Ende liegen `longmynd`, `fake_read` und `ts_analyse`
im Ordner. Bei Problemen (alte oder gemischte Objektdateien) sauber neu
bauen:

```bash
make clean
make
```

`make clean` löscht Longmynd und die `.o`-Dateien, nicht aber die gebaute
libwebsockets.

## 7. udev-Regel für den MiniTiouner

MiniTiouner abziehen, dann:

```bash
sudo cp minitiouner.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
```

MiniTiouner anstecken (USB-3.0-Port, blau) und prüfen:

```bash
lsusb | grep 0403
lsusb -v -d 0403:6010 2>/dev/null | grep iProduct
```

Erwartet: `Future Technology Devices International ... FT2232C/D/H` und als
Produktname `MiniTiouner-Express` (oder `USB <-> NIM tuner`). Damit läuft
Longmynd ohne `sudo`.

## 8. Testlauf von Hand

Stromversorgung: gutes 5V/3A-Netzteil, keine weiteren stromhungrigen
USB-Geräte am selben Port-Paar.

Als Testsignal dient die QO-100-Wideband-Bake bei **10491,5 MHz** mit
**1500 kS/s**. Longmynd bekommt die Zwischenfrequenz (ZF), je nach LO:

| LO deines LNB | ZF = 10491,5 − LO | Startwert |
|---|---|---|
| 9360 MHz | 1131,5 MHz | `1131500` (kHz) |
| 9750 MHz | 741,5 MHz | `741500` (kHz) |

Beispiel mit LO 9360 MHz (wie im Setup des Autors):

```bash
cd ~/longmynd
./longmynd -W 8080 -i 127.0.0.1 4003 -I 127.0.0.1 4002 1131500 1500
```

Mit LO 9750 MHz lautet die letzte Zeile entsprechend `... 741500 1500`.

Im Log muss in der Konfiguration stehen:

```
Web interface enabled: Port 8080
```

Dazu `STV0910 MID = 0x51, DID = 0x20` und `found new NIM with LNAs`.
Die Meldung `No-data timeout, re-init config` ist normal, solange kein
passendes Signal anliegt.

Vom PC testen: Browser `http://<RPi-IP>:8080` oder
`Test-NetConnection <RPi-IP> -Port 8080` in PowerShell.

Mit `Strg+C` beenden.

### Bedeutung der Parameter

| Teil | Bedeutung |
|---|---|
| `-W 8080` | Web-UI und WebSocket-Steuerung auf Port 8080 |
| `-i 127.0.0.1 4003` | Start-Ziel des Transportstreams (Video, UDP). Client/OpenTuner überschreiben es per WebSocket. |
| `-I 127.0.0.1 4002` | Ziel des UDP-Status. Dein Client nutzt ihn nicht (Status kommt per WebSocket), er verhindert aber den FIFO-Fehler. |
| `1131500` | Start-ZF in **kHz** (Beispiel: Bake 10491,5 MHz, LO 9360 MHz) |
| `1500` | Start-Symbolrate in **kS/s** |

Weitere Optionen: `-w` unterer NIM-Eingang, `-p v|h` LNB-Spannung (13 V / 18 V),
`-b` MER-Beep. Alle in `man -l longmynd.1`.

> **Warum `-i` und `-I` nicht weglassen?** Ohne `-I` schreibt Longmynd den
> Status in die FIFO `longmynd_main_status`. Die Datei muss dann existieren,
> sonst bricht der Start ab (`ERROR: Failed to open fifo`). Dasselbe gilt für
> `-i` und `longmynd_main_ts`. Mit `127.0.0.1` gibt es keinen Fehlerfall.
> Die FIFO wird nicht-blockierend geöffnet, ein Leser ist nicht nötig.
>
> `-i` und `-I` dürfen nicht dieselbe IP **und** denselben Port haben
> (`main.c`: *Cannot set Status IP & Port identical to TS IP & Port*).

### Frequenz (ZF) berechnen

Longmynd bekommt die Zwischenfrequenz, nicht die Empfangsfrequenz:

```
ZF (kHz) = Empfangsfrequenz (kHz) − LO (kHz)
```

Beispiele mit der Bake bei 10491,5 MHz:

- LO 9360 MHz: `10491,5 − 9360 = 1131,5 MHz = 1131500 kHz`
- LO 9750 MHz: `10491,5 − 9750 = 741,5 MHz = 741500 kHz`

Der Longmynd Client und das Web-UI
rechnen das mit „Freq Offset" (`9360000`) selbst um. Der Tuner (STV6120)
deckt etwa 144–2450 MHz ab.

## 9. Autostart per systemd

Der Dienst läuft unter dem Benutzer `dh1rk`. Die Vorlage
`longmynd.service.skel` im Repo passt nicht (Benutzer `phil`, Skript
`run.sh` fehlt), daher wird die Datei direkt angelegt:

```bash
sudo systemctl stop longmynd 2>/dev/null   # falls schon vorhanden

sudo tee /etc/systemd/system/longmynd.service > /dev/null <<'EOF'
[Unit]
Description=Longmynd Receiver
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=dh1rk
WorkingDirectory=/home/dh1rk/longmynd
ExecStart=/home/dh1rk/longmynd/longmynd -W 8080 -i 127.0.0.1 4003 -I 127.0.0.1 4002 1131500 1500
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now longmynd
```

Vorher ein von Hand gestartetes Longmynd beenden (`pgrep -a longmynd`),
sonst blockiert es MiniTiouner und Port 8080.

**Startwerte anpassen:** `1131500 1500` am Ende der `ExecStart`-Zeile sind
ZF (kHz) und Symbolrate (kS/s) der Bake bei **LO 9360 MHz**. Hat dein LNB
**LO 9750 MHz**, setze statt dessen `741500 1500` (siehe Abschnitt 8).
Die Werte gelten nur für den Start, Client oder OpenTuner stellen die
Frequenz später selbst um. Nach einer Änderung:
`sudo systemctl daemon-reload && sudo systemctl restart longmynd`.
Ist der Schreibschutz aus Abschnitt 11 schon aktiv, geht die Änderung nach
einem Neustart verloren. Schalte ihn dann zuerst ab (11.5), ändere die Datei
und aktiviere ihn wieder.

Prüfen:

```bash
systemctl status longmynd
journalctl -u longmynd -n 30 --no-pager
ss -ltn | grep 8080
```

Endtest: `sudo reboot`. Nach etwa einer Minute muss das Web-UI ohne weiteres
Zutun erreichbar sein.

## 10. Update-Ablauf

```bash
cd ~/longmynd
git pull --recurse-submodules
make
sudo systemctl restart longmynd
```

Der Pi holt Updates aus dem Fork `DH1RK/longmynd` (Branch `main`). Neuerungen
von Phil (`philcrump/longmynd`, Branch `master`) gelangen erst dorthin, wenn
sie im Fork nachgezogen wurden. Nach dem Update lässt sich mit
`grep -c json_object_new_int web/web.c` prüfen, dass der Fix von dl1rf noch
enthalten ist.

## 11. Nach erfolgreichem Aufsetzen des Longmynd-Servers: Dateisystem schützen

Für den abgesetzten Betrieb wird der Pi nur ein- und ausgeschaltet, ohne
sauberes Herunterfahren. Damit die SD-Karte dabei nicht beschädigt wird,
macht man das Dateisystem schreibgeschützt. Dafür dient das Overlay-Dateisystem
von Raspberry Pi OS: Root und Boot werden nur gelesen, alle Schreibzugriffe
landen im RAM und sind nach dem Ausschalten weg.

**Erst aktivieren, wenn alles fertig ist** (Schritte 1–10 abgeschlossen,
Autostart per Reboot-Test bestätigt). Spätere Änderungen erfordern, den
Schutz vorher abzuschalten (siehe unten).

### 11.1 Overlay aktivieren

Auf dem Pi, wahlweise über das Menü (Variante a) oder direkt per Befehl
(Variante b).

**Variante a: über `raspi-config`**

```bash
sudo raspi-config
```

`Performance Options` → `Overlay File System` → *Enable overlay file system*
mit **Yes**, und die Frage nach der schreibgeschützten Boot-Partition ebenfalls
mit **Yes** beantworten. Danach neu starten.

**Variante b: direkte Befehle**

```bash
sudo raspi-config nonint enable_overlayfs
sudo raspi-config nonint enable_bootro
sudo reboot
```

### 11.2 Prüfen

```bash
mount | grep -E "overlay|root-ro|root-rw"
findmnt /boot/firmware
```

Erwartet (gekürzt):

```
tmpfs-root /media/root-rw tmpfs rw,relatime 0 0
overlayroot / overlay rw,relatime,lowerdir=/media/root-ro,upperdir=/media/root-rw/overlay,... 0 0
/dev/mmcblk0p2 /media/root-ro ext4 ro,relatime 0 0
```

und für `/boot/firmware` in der Spalte `OPTIONS` ein `ro`. Die SD-Karte
(`/dev/mmcblk0p2`) ist `ro` eingehängt, die obere Schicht des Overlays liegt
auf einem `tmpfs` im RAM.

### 11.3 Funktionstest

`/` gehört root, daher `sudo` verwenden (ohne `sudo` kommt
`Permission denied`, unabhängig vom Overlay):

```bash
sudo touch /testdatei && ls /testdatei
ls /media/root-rw/overlay/testdatei     # liegt im RAM (Overlay, obere Schicht)
ls /media/root-ro/testdatei             # fehlt: nichts wurde auf die SD-Karte geschrieben
sudo reboot
```

Nach dem Neustart meldet `ls /testdatei` *No such file or directory*. Dann
ist der Schutz bestätigt. Prüfe außerdem, dass Longmynd und das Web-UI nach dem
Neustart von selbst laufen (`systemctl status longmynd`, Browser auf Port 8080).
Danach darf der Pi jederzeit einfach vom Strom getrennt werden.

### 11.4 Hinweise für den Betrieb

- **Einstellungen sind nicht dauerhaft.** Frequenz, Symbolrate und alles, was
  über das Web-UI oder den Client verstellt wird, gilt nur bis zum Ausschalten.
  Startwerte stehen in der `ExecStart`-Zeile des Dienstes (Schritt 9).
- **Logs gehen verloren.** Sie liegen im RAM. Damit sie den Arbeitsspeicher
  nicht füllen, kann das Journal begrenzt werden (`/etc/systemd/journald.conf`:
  `Storage=volatile`, `RuntimeMaxUse=20M`).
- **Uhrzeit:** Der Pi hat keine Echtzeituhr und stellt die Zeit erst per NTP.
  Das betrifft nur Zeitstempel in Logs, nicht den Empfang.
- **Dauerhaft speichern** (falls nötig): eigene kleine Daten-Partition oder
  USB-Stick mit `rw,noatime` einhängen und nur dorthin schreiben.

### 11.5 Änderungen und Updates am Server

Für Updates (Schritt 10), neue Pakete oder geänderte Dienste muss der Schutz
vorübergehend aus.

**Variante a: über `raspi-config`**

`sudo raspi-config` → `Performance Options` → `Overlay File System` →
*Disable overlay file system* (und die Boot-Partition wieder beschreibbar
machen), dann `sudo reboot`. Nach den Änderungen auf demselben Weg wieder
aktivieren (siehe 11.1 Variante a).

**Variante b: direkte Befehle**

```bash
sudo raspi-config nonint disable_overlayfs
sudo raspi-config nonint disable_bootro
sudo reboot
```

Dann Änderungen vornehmen und testen, anschließend wieder einschalten:

```bash
sudo raspi-config nonint enable_overlayfs
sudo raspi-config nonint enable_bootro
sudo reboot
```

## Weiter mit der Inbetriebnahme

Client-Einstellungen, Fehlersuche und Merkpunkte stehen in
[BringingUp.md](BringingUp.md).
