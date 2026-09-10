# Backup-Server einrichten (alle 6 h Vollsicherung)

Der neue Server (2 vCPU / 2 GB / 45 GB SSD, anderer Anbieter) wird zum zentralen
Backup-Server. Er zieht alle 6 Stunden die Datenbank plus alle Projektdateien von
Portal, Backend, Landing, Bot und WebID-Server und legt sie als Archiv ab.
Im Notfall bestellen wir neue Server und spielen das letzte Archiv zurück
(Wiederanlauf ca. 4–8 h, maximaler Datenverlust 6 h).

```text
Portal .124 ──┐
Backend .123 ─┼── alle 6 h per SSH/RSYNC ──▶ Backup-Server (neu, anderer Anbieter)
Landing ──────┘                              /var/backups/portal/{daily,monthly,logs}
Bot, WebID ───┘        Status erscheint im Portal unter Admin → Infrastruktur
```

## Ablauf (schrittweise zusammen)

1. **IP des Backup-Servers mitteilen** — brauchen wir für die Konfiguration.
2. Auf dem Backup-Server: Repo klonen (oder nur die Skripte hochladen) und
   `bash scripts/setup-backup-server.sh` ausführen. Das legt den Ordner
   `/var/backups/portal/{daily,monthly,logs}` an.
3. SSH-Schlüssel: Auf dem Backup-Server einen Schlüssel erzeugen und dessen
   öffentlichen Teil auf Portal-, Backend-, Landing-, Bot- und WebID-Server in
   `/root/.ssh/authorized_keys` eintragen, damit der Backup-Server die Daten
   abholen darf.
4. `scripts/backup-orchestrator.env` anlegen (Vorlage liegt bei) und eintragen:
   Backend-IP, die übrigen Server-IPs, Aufbewahrung 14 Tage.
5. `bash scripts/install-backup-orchestrator.sh` — richtet den automatischen
   Lauf alle 6 h ein (0:00, 6:00, 12:00, 18:00 Uhr).
6. Erstlauf starten: `bash scripts/backup-orchestrator.sh full` und prüfen, ob
   unter `/var/backups/portal/daily/` ein Archiv liegt. Danach im Portal unter
   **Admin → Infrastruktur** kontrollieren, ob der Backup-Status auftaucht.
7. Optional, empfohlen: Verschlüsselung der Archive mit `age` aktivieren,
   privaten Schlüssel in deinen Passwortmanager legen.

## Was gesichert wird

- Komplette Datenbank (pg_dump vom Backend-Server)
- Portal-Code und Supabase-Konfiguration (Backend)
- Landing-Server-Dateien (inkl. Caddyfile und Domains)
- Bot-Server und WebID-Server

Aufbewahrung: tägliche Archive 14 Tage, zusätzlich jeweils am 1. des Monats ein
Monatsarchiv für 90 Tage.

## Technischer Hintergrund

- Keine Code-Änderungen nötig — alle Skripte existieren bereits:
  `scripts/setup-backup-server.sh`, `scripts/backup-orchestrator.sh`,
  `scripts/install-backup-orchestrator.sh`, `scripts/restore.sh`,
  Timer-Units unter `scripts/systemd/`.
- Der Orchestrator läuft per systemd-Timer auf dem Backup-Server als root und
  meldet jeden Lauf in die Tabelle `backup_status` (sichtbar im Portal).
- Der Notfall-Ablauf (neue Server + Restore) ist in
  `docs/DISASTER-RECOVERY.md` dokumentiert.

## Voraussetzung vom Nutzer

- IP-Adresse des neuen Backup-Servers (Schritt 1)
- Root-SSH-Zugang zu allen fünf Produktions-Servern (vorhanden)
