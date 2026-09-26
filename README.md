# USBGuard Notify

Ein kleiner Desktop-Agent für **USBGuard** unter Linux Mint / Cinnamon.

Neue und noch nicht bekannte USB-Geräte werden zunächst durch USBGuard
blockiert. Anschließend erscheint ein Desktop-Dialog, über den das Gerät
blockiert, einmalig freigegeben oder dauerhaft angelernt werden kann.

Gedacht ist das insbesondere als zusätzliche Schutzmaßnahme gegen
**BadUSB-, HID-Injection- und O.MG-Cable-Angriffe**.

<img width="490" height="524" alt="image" src="https://github.com/user-attachments/assets/fcd2984e-e3ba-48aa-8ed9-63abc387be6b" />


## Funktionsweise

USBGuard übernimmt die eigentliche Kontrolle der USB-Geräte.

Die Policy ist so konfiguriert, dass unbekannte Geräte standardmäßig
blockiert werden:

```ini
ImplicitPolicyTarget=block
InsertedDevicePolicy=apply-policy
```

`usbguard-notify` überwacht anschließend die IPC-Events von USBGuard:

```bash
usbguard watch
```

Wird ein neues Gerät mit

```text
event=Insert
target=block
```

erkannt, erscheint ein Zenity-Dialog.

Der Benutzer kann zwischen drei Aktionen wählen:

| Aktion | Verhalten |
|---|---|
| **Blockieren** | Das Gerät bleibt blockiert |
| **Einmal erlauben** | Gerät wird für die aktuelle Verbindung freigegeben |
| **Dauerhaft erlauben** | Eine permanente USBGuard-Regel wird angelegt |

`Enter`, `ESC` oder das Schließen des Fensters lassen das Gerät
standardmäßig blockiert.

## HID-Warnung

USBGuard Notify wertet die USB-Interface-Klassen des Gerätes aus.

Wird eine HID-Schnittstelle der USB-Klasse `03` erkannt, erscheint
zusätzlich eine Warnung:

```text
⚠ WARNUNG: HID-Schnittstelle erkannt!
Dieses Gerät kann Tastatur- oder Mauseingaben erzeugen.
```

Beispiele:

```text
03:01:01    HID-Tastatur
03:01:02    HID-Maus
03:00:00    HID-Gerät
08:06:50    Massenspeicher
09:xx:xx    USB-Hub
01:xx:xx    Audio
0e:xx:xx    Video/Kamera
```

Damit ist beispielsweise ein vermeintlicher USB-Datenträger auffällig,
wenn er zusätzlich ein HID-Keyboard anmeldet.

## Voraussetzungen

Getestet unter:

- Linux Mint Cinnamon
- USBGuard
- Zenity

Benötigte Pakete:

```bash
sudo apt install usbguard zenity
```

`flock` wird ebenfalls verwendet und ist normalerweise Bestandteil von
`util-linux`.

## USBGuard konfigurieren

Vor dem Aktivieren von USBGuard sollte zunächst eine Policy für die
bereits angeschlossenen und vertrauenswürdigen Geräte erzeugt werden.

```bash
sudo usbguard generate-policy > /tmp/usbguard-rules.conf
```

Policy kontrollieren:

```bash
cat /tmp/usbguard-rules.conf
```

Anschließend übernehmen:

```bash
sudo install \
    -m 0600 \
    -o root \
    -g root \
    /tmp/usbguard-rules.conf \
    /etc/usbguard/rules.conf
```

Wichtige Einstellungen in:

```text
/etc/usbguard/usbguard-daemon.conf
```

sind beispielsweise:

```ini
RuleFile=/etc/usbguard/rules.conf

ImplicitPolicyTarget=block
PresentDevicePolicy=apply-policy
InsertedDevicePolicy=apply-policy

IPCAllowedUsers=root
IPCAllowedGroups=root plugdev
```

USBGuard danach aktivieren:

```bash
sudo systemctl enable --now usbguard
```

## Benutzerberechtigung

Der Desktop-Benutzer muss auf die USBGuard-IPC-Schnittstelle zugreifen
dürfen.

Bei einer Konfiguration über die Gruppe `plugdev` kann geprüft werden:

```bash
id
```

Die Ausgabe sollte `plugdev` enthalten.

Falls nicht:

```bash
sudo usermod -aG plugdev "$USER"
```

Danach einmal ab- und wieder anmelden.

Alternativ können die USBGuard-eigenen IPC-ACLs verwendet werden.

## IPC testen

USBGuard muss sich ohne `sudo` abfragen lassen:

```bash
usbguard list-devices
```

Danach:

```bash
usbguard watch
```

Beim Einstecken eines unbekannten Gerätes sollte beispielsweise
erscheinen:

```text
[device] PresenceChanged: id=50
 event=Insert
 target=block
 device_rule=block id 04c5:2028 \
 serial "..." \
 name "iodd2531" \
 via-port "4-3" \
 with-interface 08:06:50
```

## Installation

Repository klonen oder das Script nach

```text
~/.local/bin/usbguard-notify
```

kopieren.

Anschließend ausführbar machen:

```bash
chmod 700 ~/.local/bin/usbguard-notify
```

Manueller Test:

```bash
~/.local/bin/usbguard-notify
```

Das Script wartet anschließend auf USBGuard-Events.

Ein unbekanntes USB-Gerät einstecken.

USBGuard sollte das Gerät zunächst blockieren und der Dialog sollte
automatisch erscheinen.

## Autostart unter Cinnamon

Autostart-Verzeichnis anlegen:

```bash
mkdir -p ~/.config/autostart
```

Datei erstellen:

```text
~/.config/autostart/usbguard-notify.desktop
```

Inhalt:

```ini
[Desktop Entry]
Type=Application
Name=USBGuard Notify
Comment=USBGuard Desktop-Abfrage für neue USB-Geräte
Exec=/home/USER/.local/bin/usbguard-notify
Terminal=false
X-GNOME-Autostart-enabled=true
```

`USER` durch den jeweiligen Benutzernamen ersetzen.

Nach der nächsten Desktop-Anmeldung wird `usbguard-notify`
automatisch gestartet.

## Einzelinstanz

Das Script verwendet `flock`, damit immer nur eine Instanz läuft.

Dadurch führt beispielsweise

```bash
~/.local/bin/usbguard-notify
```

nicht zu einer zweiten Instanz, wenn der Agent bereits über den
Desktop-Autostart gestartet wurde.

Prüfen:

```bash
pgrep -af usbguard-notify
```

## Temporäres Freigeben

Die Aktion

```text
Einmal erlauben
```

führt intern aus:

```bash
usbguard allow-device DEVICE_ID
```

Die Entscheidung wird nicht dauerhaft in die USBGuard-Policy
geschrieben.

Wird das Gerät später erneut eingesteckt, wird es wieder blockiert und
erneut abgefragt.

## Dauerhaftes Freigeben

Die Aktion

```text
Dauerhaft erlauben
```

führt aus:

```bash
usbguard allow-device -p DEVICE_ID
```

USBGuard fügt dadurch eine permanente Allow-Regel zur Policy hinzu.

Die Regeln können angezeigt werden mit:

```bash
usbguard list-rules
```

Beispielsweise:

```bash
usbguard list-rules | grep -i iodd
```

## Geräte anzeigen

Alle bekannten Geräte:

```bash
usbguard list-devices
```

Nur erlaubte Geräte:

```bash
usbguard list-devices --allowed
```

Nur blockierte Geräte:

```bash
usbguard list-devices --blocked
```

## Logging

USBGuard kann über das systemd-Journal überwacht werden:

```bash
sudo journalctl -u usbguard
```

Live:

```bash
sudo journalctl -u usbguard -f
```

Wenn der FileAudit-Backend aktiviert ist:

```ini
AuditBackend=FileAudit
AuditFilePath=/var/log/usbguard/usbguard-audit.log
```

kann das Audit-Log beispielsweise mit

```bash
sudo tail -f /var/log/usbguard/usbguard-audit.log
```

beobachtet werden.

Dort werden unter anderem folgende Ereignisse protokolliert:

```text
Device.Insert
Device.Remove
Policy.Device.Update
```

sowie der jeweilige Zustand:

```text
target.new='allow'
target.new='block'
```

## Sicherheitsmodell

USBGuard Notify selbst blockiert keine USB-Geräte.

Die Sicherheitsentscheidung erfolgt bereits vorher durch USBGuard:

```text
USB-Gerät
    │
    ▼
Linux Kernel
    │
    ▼
USBGuard
    │
    ├── bekannte Policy → allow
    │
    └── unbekannt       → block
                              │
                              ▼
                       usbguard-notify
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                  Block    einmal    dauerhaft
                           erlauben    erlauben
```

Dadurch ist das neue Gerät bereits blockiert, bevor der Benutzer den
Desktop-Dialog bestätigt.

Das ist insbesondere bei HID-Injection wichtig: Ein neu angeschlossenes
Keyboard sollte nicht erst Tastatureingaben ausführen können und
anschließend vom Desktop-Agenten blockiert werden.

## O.MG Cable / BadUSB

Ein O.MG Cable oder anderes BadUSB-Gerät kann sich gegenüber dem
Betriebssystem unter anderem als USB-HID-Gerät präsentieren.

Ein typisches HID-Keyboard besitzt beispielsweise:

```text
03:01:01
```

USBGuard Notify markiert deshalb alle Schnittstellen mit USB-Klasse

```text
03
```

besonders.

Dabei handelt es sich jedoch nicht um eine automatische Erkennung eines
bestimmten Angriffswerkzeugs.

USBGuard kann lediglich die vom USB-Gerät präsentierten Eigenschaften
und Interfaces kontrollieren.

## Beispiel

Ein USB-Massenspeicher könnte beispielsweise erscheinen als:

```text
Name: iodd2531
USB-ID: 04c5:2028
Seriennummer: XXXXXXXX
Port: 4-3

Schnittstellen:
Massenspeicher (08:06:50)
```

Ein Gerät mit zusätzlicher HID-Funktion könnte dagegen zeigen:

```text
Schnittstellen:

Massenspeicher (08:06:50)
HID-Tastatur (03:01:01)

⚠ WARNUNG: HID-Schnittstelle erkannt!
Dieses Gerät kann Tastatur- oder Mauseingaben erzeugen.
```

## Hinweise

Die initiale USBGuard-Policy sollte sorgfältig geprüft werden.

Insbesondere fest eingebaute Geräte wie

- interne Kamera
- Bluetooth-Controller
- Fingerprint-Reader
- Dockingstation
- interne USB-Hubs

sollten bereits vor Aktivierung der restriktiven Policy berücksichtigt
werden.

Andernfalls können diese Geräte nach dem Start von USBGuard blockiert
werden.

## Projektdateien

```text
usbguard-notify/
├── README.md
├── usbguard-notify
└── docs/
    └── usbguard-notify.png
```

## Lizenz

Eine Lizenz kann je nach gewünschter Veröffentlichung ergänzt werden.
Für ein kleines Open-Source-Projekt bietet sich beispielsweise die
MIT-Lizenz an.
