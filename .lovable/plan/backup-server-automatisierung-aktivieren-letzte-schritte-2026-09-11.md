# Backup-Server: Automatisierung aktivieren (letzte Schritte)

Der Erstlauf ist erfolgreich (Archiv 854 MB, alle 5 Server gesichert). Jetzt wird
der automatische 6-Stunden-Lauf eingerichtet — danach sind keine manuellen Läufe
mehr nötig.

```text
Systemd-Timer auf dem Backup-Server:  0:00 / 6:00 / 12:00 / 18:00 Uhr
Ziel: /var/backups/portal/{daily,monthly,logs}
Status im Portal: Admin → Infrastruktur
```

## Schritt 1 — Timer aktivieren (auf dem Backup-Server als root)

```bash
cd /opt/apps/portal
bash scripts/install-backup-orchestrator.sh
```

Das Skript installiert rsync/ssh (falls nötig), kopiert die systemd-Units und
aktiviert den Timer. Erwartete Ausgabe: "Units installiert", "Timer aktiviert"
und ein Status mit `active (waiting)`.

## Schritt 2 — Prüfen

```bash
systemctl list-timers backup-orchestrator.timer
```

Erwartet: Zeile mit `backup-orchestrator.timer` und angabe, wann der nächste
Lauf startet (z. B. "in 3 h").

## Schritt 3 — Backup-Status im Portal kontrollieren

Im Portal unter **Admin → Infrastruktur** sollte nach dem nächsten automatischen
Lauf ein Status-Eintrag erscheinen (der Orchestrator schreibt jeden Lauf in die
Tabelle `backup_status`).

## Schritt 4 (optional, empfohlen) — Verschlüsselung mit age

```bash
# 1. age installieren (auf dem Backup-Server)
apt-get install -y age

# 2. Schlüsselpaar erzeugen
age-keygen -o /root/age.key

# 3. Öffentlichen Schlüssel anzeigen (beginnt mit age1...)
cat /root/age.key | grep public
```

Den öffentlichen Schlüssel in `scripts/backup-orchestrator.env` ergänzen und den
privaten Schlüssel (`/root/age.key`) in den Passwortmanager kopieren — er wird
nur beim Restore gebraucht und darf den Server nicht verlassen.

## Keine Code-Änderungen nötig

Alle Skripte und Units existieren bereits im Repo. Nach Schritt 1–3 ist das
Backup-System komplett: Vollsicherung alle 6 h, 14 Tage Aufbewahrung, Monatsarchive
90 Tage, Restore laut `docs/DISASTER-RECOVERY.md`.

## Voraussetzung vom Nutzer

- Root-SSH-Zugang zum Backup-Server (vorhanden)
- Für Schritt 4: Bereitschaft, den privaten Schlüssel im Passwortmanager abzulegen
