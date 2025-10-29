# systemd Notes

Kurze, praxisnahe Notizen zu **systemd**: Services verwalten, eigene Unit-Dateien bauen, Timer (Cron-Ersatz), Logs mit `journalctl`, Drop-ins/Overrides, User-Services, Hardening & Diagnose.

**Inhalt:**  
[Quickstart](#quickstart) · [Services managen](#services) · [Service-Units (Beispiele)](#service-units) · [User-Services](#user-services) · [Timer (Beispiele)](#timer) · [Logs / journalctl](#journalctl) · [Drop-ins & Reload](#dropins) · [Abhängigkeiten & Boot](#deps) · [Environment & Dateien](#env) · [Hardening](#hardening) · [Nützliche Einzeiler](#one-liners) · [Troubleshooting](#troubleshooting)

---

<a id="quickstart"></a>
## 🚀 Quickstart

```bash
# Status & Liste
systemctl status
systemctl list-units --type=service --state=running

# Service steuern
sudo systemctl start NAME.service
sudo systemctl stop NAME.service
sudo systemctl restart NAME.service
sudo systemctl reload NAME.service      # nur Konfig neu laden (falls unterstützt)

# Autostart
sudo systemctl enable NAME.service
sudo systemctl disable NAME.service

# Zustand prüfen
systemctl is-active NAME.service
systemctl is-enabled NAME.service

# Änderungen an Unit-Dateien einlesen
sudo systemctl daemon-reload

# Logs einer Unit
journalctl -u NAME.service -e -f
```

---

<a id="services"></a>
## 🧭 Services managen

```bash
# Fehlgeschlagene Units
systemctl --failed

# Abhängigkeiten / Startreihenfolge
systemctl list-dependencies NAME.service

# Effektive Unit (inkl. Drop-ins) anzeigen
systemctl cat NAME.service

# Override interaktiv anlegen/ändern (Drop-in)
sudo systemctl edit NAME.service

# Temporärer Einmal-Job als eigene Unit
systemd-run --unit=temp-demo --property=Description="Temp Demo" /usr/bin/env echo "hi"
```

---

<a id="service-units"></a>
## 🧩 Service-Units (Beispiele)

**Skeleton / Vorlage:**
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=YOURUSER
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/run.sh
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production
# Alternative: EnvironmentFile=/etc/myapp.env
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**One-shot Job (einmalige Aktion):**
```ini
# /etc/systemd/system/db-backup.service
[Unit]
Description=DB Backup (oneshot)

[Service]
Type=oneshot
User=YOURUSER
ExecStart=/usr/local/bin/db-backup.sh
RemainAfterExit=no

[Install]
WantedBy=multi-user.target
```

> Nach Änderungen: `sudo systemctl daemon-reload && sudo systemctl enable --now myapp.service`

---

<a id="user-services"></a>
## 👤 User-Services (ohne Root)

Pfad: `~/.config/systemd/user/`

```ini
# ~/.config/systemd/user/notes-sync.service
[Unit]
Description=Sync Notes (user service)

[Service]
Type=oneshot
ExecStart=/home/USER/bin/notes-sync.sh
```

Aktivieren/Starten:

```bash
systemctl --user daemon-reload
systemctl --user enable --now notes-sync.service

# User-Services auch ohne Login am Laufen halten:
sudo loginctl enable-linger USER
```

---

<a id="timer"></a>
## ⏰ Timer (Cron-Ersatz) – Beispiele

**1) Nightly Backup 03:15 Uhr**
```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Nightly rsync Backup

[Service]
Type=oneshot
User=YOURUSER
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup.service nightly at 03:15

[Timer]
OnCalendar=*-*-* 03:15:00
Persistent=true           # Nachholen, falls verpasst (Gerät war aus)
Unit=backup.service

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers --all
```

**2) Alle 15 Minuten**
```ini
# /etc/systemd/system/pulse-job.service
[Unit]
Description=Pulse Job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/pulse.sh
```

```ini
# /etc/systemd/system/pulse-job.timer
[Unit]
Description=Pulse alle 15 Minuten

[Timer]
OnUnitActiveSec=15m
Unit=pulse-job.service

[Install]
WantedBy=timers.target
```

**3) Beim Boot + danach alle 10 Min**
```ini
# /etc/systemd/system/refresh.timer
[Unit]
Description=Refresh @boot und alle 10m

[Timer]
OnBootSec=2min
OnUnitActiveSec=10min
Unit=refresh.service

[Install]
WantedBy=timers.target
```

---

<a id="journalctl"></a>
## 📜 Logs / `journalctl`

```bash
# Live folgen
journalctl -f

# Unit-spezifisch (neueste zuerst, zum Ende springen)
journalctl -u myapp.service -e

# Zeitfenster
journalctl --since "today" --until "now"
journalctl --since "2025-10-01 08:00" -u myapp.service

# Nur aktueller Boot
journalctl -b -u myapp.service

# Nach Priorität (0=emerg … 7=debug)
journalctl -p warning -u myapp.service

# Mit grep
journalctl -u myapp.service | grep -i error
```

**Persistente Logs aktivieren (Ubuntu/Mint):**
```bash
sudo mkdir -p /var/log/journal
sudo sed -ri 's/^#?Storage=.*/Storage=persistent/' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
```

Optional in `/etc/systemd/journald.conf`:
```
SystemMaxUse=200M
SystemMaxFileSize=20M
```

---

<a id="dropins"></a>
## 🧩 Drop-ins, Overrides & Reload

```bash
# Override-Datei (Drop-in) erstellen:
sudo systemctl edit NAME.service

# Beispiel-Inhalt:
# [Service]
# Environment=APP_ENV=prod
# Restart=always

# Änderungen einlesen + neu starten:
sudo systemctl daemon-reload
sudo systemctl restart NAME.service

# Effektive Unit (inkl. Drop-ins) anzeigen:
systemctl cat NAME.service
```

---

<a id="deps"></a>
## 🔗 Abhängigkeiten & Boot

```ini
[Unit]
# Startet NACH Netzwerk (IP konfiguriert)
After=network-online.target
Wants=network-online.target

# Harte Abhängigkeit:
Requires=postgresql.service
After=postgresql.service
```

**Targets (häufig):**
- `multi-user.target` (Server/CLI, „Runlevel 3“)
- `graphical.target` (Desktop, „Runlevel 5“)
- `timers.target` (Sammelziel für Timer)

---

<a id="env"></a>
## 🗂️ Environment & Dateien

**EnvironmentFile verwenden:**
```ini
# /etc/systemd/system/myapp.service
[Service]
EnvironmentFile=/etc/myapp.env
ExecStart=/opt/myapp/run.sh
```

```bash
# /etc/myapp.env
APP_PORT=8080
APP_MODE=production
```

**WorkingDirectory & Pfade:**
```ini
[Service]
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 app.py
```

---

<a id="hardening"></a>
## 🛡️ Hardening (schnelle Wins)

```ini
# In der [Service]-Sektion
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full       # macht /usr /boot /etc read-only
ProtectHome=read-only    # $HOME read-only mounten
ProtectKernelLogs=true
ProtectControlGroups=true
LockPersonality=true
CapabilityBoundingSet=
AmbientCapabilities=
# Falls Schreibpfade nötig sind:
# ReadWritePaths=/opt/myapp/data
```

---

<a id="one-liners"></a>
## 🧰 Nützliche Einzeiler

```bash
# Alle Timer mit nächster Ausführung
systemctl list-timers

# Service + Timer-Skelett anlegen
NAME=demo && sudo bash -c "cat >/etc/systemd/system/$NAME.service" <<'EOF'
[Unit]
Description=Demo oneshot
[Service]
Type=oneshot
ExecStart=/usr/bin/env echo hello
EOF

sudo bash -c "cat >/etc/systemd/system/$NAME.timer" <<'EOF'
[Unit]
Description=Demo Timer
[Timer]
OnUnitActiveSec=5min
Unit=demo.service
[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload && sudo systemctl enable --now "$NAME.timer"

# Journald: nur Fehler der letzten 1h
journalctl --since "1 hour ago" -p err

# Unit hart killen (inkl. Children)
sudo systemctl kill -s SIGKILL NAME.service
```

---

<a id="troubleshooting"></a>
## 🧯 Troubleshooting

- **Service startet nicht:**  
  `systemctl status NAME -l` und `journalctl -u NAME -b -e` prüfen. Danach oft vergessen: `sudo systemctl daemon-reload`.

- **Environment wird nicht geladen:**  
  Pfad in `EnvironmentFile=` prüfen; Datei im Format `KEY=value`.

- **Netzwerk noch nicht bereit:**  
  `After=`/`Wants=network-online.target` setzen und `*-wait-online` benutzen.

- **Timer „verpasst“ während Gerät aus:**  
  In `.timer`: `Persistent=true`.

- **User-Service stoppt nach Logout:**  
  `sudo loginctl enable-linger USER`.

---

> **Hinweis:** Platzhalter (`YOURUSER`, `/opt/myapp`, etc.) an dein System anpassen.