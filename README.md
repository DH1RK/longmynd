# Longmynd Server for Raspberry Pi (Fork)

Longmynd is an open source Linux ATV receiver for the MiniTiouner
(Copyright 2019 Heather Lomond). This fork is based on
[philcrump/longmynd](https://github.com/philcrump/longmynd) (web UI and
WebSocket control) and additionally contains the fix from
[PR #2](https://github.com/philcrump/longmynd/pull/2) by dl1rf (integer
values in the WebSocket JSON), so that the Longmynd Client (ZR6TG) and
OpenTuner show a usable status.

Longmynd ist ein quelloffener Linux-ATV-Empfänger für den MiniTiouner. Dieser
Fork basiert auf `philcrump/longmynd` (Web-UI, WebSocket) und enthält den Fix
aus PR #2 von dl1rf (Ganzzahlen im JSON), damit Longmynd Client und OpenTuner
einen brauchbaren Status anzeigen.

Getestet mit: Raspberry Pi 4B (64-Bit), MiniTiouner Express, Raspberry Pi OS
(64-bit), Raspberry Pi Imager v2.0.12.

## Dokumentation / Documentation

| Schritt | Inhalt | Dokument |
|---|---|---|
| 1 | Raspberry-Pi-Image erstellen (Imager) | [01_RaspberryPi_Imager.md](docs/01_RaspberryPi_Imager.md) (DE), [01_RaspberryPi_Imager_EN.md](docs/01_RaspberryPi_Imager_EN.md) (EN) |
| 2 | Longmynd installieren (Pi einrichten, Quellcode, Autostart, Schreibschutz) | [HowTo_Install.md](docs/HowTo_Install.md) (DE), [HowTo_Install_EN.md](docs/HowTo_Install_EN.md) (EN) |
| 3 | Inbetriebnahme, Longmynd Client und OpenTuner, Fehlersuche | [BringingUp.md](docs/BringingUp.md) (DE), [BringingUp_EN.md](docs/BringingUp_EN.md) (EN) |
| 4 | Dateisystem schützen (Betrieb mit Ein/Aus am Netzteil) | Abschnitt 11 in [HowTo_Install.md](docs/HowTo_Install.md) |
| 5 | Originaldokumentation von Longmynd (Optionen, Status- und TS-Schnittstellen) | [README_Longmynd.md](docs/README_Longmynd.md) |

## Schnellüberblick

```
Raspberry Pi 4B + MiniTiouner Express (USB 3.0)
        │  WebSocket 8080 (Steuerung + Status)
        │  UDP 4003 (Video, MPEG-TS)
        ▼
PC mit Longmynd Client (ZR6TG) oder OpenTuner  →  VLC / mpv
```

```bash
git clone --recursive https://github.com/DH1RK/longmynd.git
cd longmynd
make
```

Die ausführliche Anleitung steht in den oben verlinkten Dokumenten.

## Herkunft

```
myorangedragon/longmynd   (BATC original)
└── philcrump/longmynd    (web UI and WebSocket)
    ├── dl1rf/longmynd    (fix: integers in JSON, PR #2)
    └── DH1RK/longmynd    (this fork: philcrump + PR #2)
```

## Lizenz / License

Siehe [LICENSE](LICENSE).
