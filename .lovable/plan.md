# Umzug Frontend- + Backend-Server (Portal & Supabase)

## Kurzantwort: Wie lange dauert es?

| Phase | Dauer | Ausfallzeit? |
|---|---|---|
| Neue Server bestellen + OS installieren | 0,5–2 h (je nach Provider) | nein |
| Backend parallel auf neuem Server aufbauen (Supabase-Stack + Datenbank-Restore) | 2–4 h | nein |
| Frontend parallel auf neuem Server aufbauen | 0,5–1 h | nein |
| Testlauf auf den neuen Servern (per hosts-Eintrag, ohne DNS-Änderung) | 0,5–1 h | nein |
| **Umschalten: letzter Datenbank-Dump + DNS-Wechsel in Cloudflare** | **15–60 Min** | **ja, nur hier** |
| Kontrolle + IP-Referenzen in Skripten anpassen | 1 h | nein |

**Gesamt: ca. ein Arbeitstag, davon echte Ausfallzeit nur 15–60 Minuten** am Ende
(der letzte Datenbank-Dump muss einspielen, bevor die Domain auf die neuen
Server zeigt). Da die Domains (mb-portal.com / api.mb-portal.com) bleiben,
ändern sich nur die A-Records — mit TTL 300 in Cloudflare greift das in
wenigen Minuten.

## Grundprinzip

Nicht "umziehen", sondern **nebenbei neu aufbauen und dann umschalten**:
Beide alten Server bleiben laufen, bis die neuen fertig sind. Die Datenbank ist
der einzige kritische Teil — sie wird zuletzt mit einem frischen Dump
übernommen, damit nichts verloren geht.

## Phase 0 — Backup-Server aufsetzen (zuerst, +2–3 h einmalig)

Vor dem Umzug brauchst du laufende Backups — sie sind gleichzeitig die
Grundlage für den Umzug selbst. Im Projekt ist dafür bereits alles vorhanden;
es ist nur noch nicht eingerichtet.

1. Kleinen VPS bestellen (Ubuntu 22.04/24.04, 2 vCPU, 4 GB RAM, Platte
   mindestens 5× so groß wie die Datenbank). Empfehlung: anderer Anbieter oder
   anderes Rechenzentrum als Portal/Backend.
2. Auf dem Backup-Server: `bash scripts/setup-backup-server.sh`
   (legt Backup-Nutzer, Verzeichnisse `/var/backups/portal/{daily,monthly,logs}` an).
3. Auf Portal, Backend, Landing, Bot und WebID jeweils den Zugriff erlauben:
   `ssh-copy-id -i /root/.ssh/id_rsa.pub root@<BACKUP-IP>` (bzw. Key des
   Backup-Servers dort in `authorized_keys` eintragen).
4. Auf dem Backup-Server Konfiguration anlegen:
   `cp scripts/backup-orchestrator.env.example scripts/backup-orchestrator.env`
   und `DB_HOST`, `SERVER_PORTAL_HOST`, `SERVER_LANDING_HOST`, `SERVER_BOT_HOST`,
   `SERVER_WEBID_HOST`, `BACKUP_DIR`, `BACKUP_RETENTION_DAYS` eintragen.
5. Zeitplan aktivieren: `bash scripts/install-backup-orchestrator.sh`
   (sichert automatisch alle 6 Stunden: Datenbank-Dump, Supabase-Volumes,
   Configs und alle fünf Server; hält 14 Tage täglich + 90 Tage monatlich).
6. Erstlauf und Kontrolle: `bash scripts/backup-orchestrator.sh full`,
   danach `ls -lt /var/backups/portal/daily/ | head`.
   Der Status erscheint auch im Portal unter `/admin/infrastructure`.
7. Empfohlen: Verschlüsselung aktivieren (`age-keygen`, öffentlichen Schlüssel
   in die Backup-Konfiguration, privaten Schlüssel in den Passwortmanager) und
   einmal eine Probe-Wiederherstellung auf einem Wegwerf-Server testen.

**Datenverlust im Ernstfall: maximal 6 Stunden.** Wenn du weniger willst, den
Timer in `scripts/systemd/backup-orchestrator.timer` auf stündlich stellen.

## Phase 1 — Vorbereitung für den Umzug (keine Ausfallzeit)

1. Server bestellen (Ubuntu 22.04/24.04):
   - Backend (neu): 4 vCPU, 8 GB RAM, 160 GB SSD
   - Frontend/Portal (neu): 2 vCPU, 4 GB RAM, 40 GB SSD
2. In Cloudflare TTL der A-Records (mb-portal.com, api.mb-portal.com) auf 300 senken.
3. Frisches Vollbackup ziehen (Backup-Server: `bash scripts/backup-orchestrator.sh full`)
   und das Archiv auf den neuen Backend-Server kopieren.

## Phase 2 — Backend-Server neu aufbauen (parallel, Portal läuft weiter)

1. Supabase-Stack wie auf .123 installieren (Docker + `/opt/supabase`).
   Wichtig: dieselben Secrets/Keys übernehmen (`/opt/supabase/.env`: JWT_SECRET,
   ANON/SERVICE-Keys, SMTP-Zugangsdaten), sonst brechen Sessions und Mails.
2. Repo klonen: `github.com/DianaKnodel1/direct-zip-import.git` nach `/opt/apps/portal`.
3. Datenbank-Restore aus dem Backup-Archiv:
   `bash scripts/restore.sh <archiv>.tar.gz` (spielt DB, Storage, Configs zurück).
4. Zertifikate/TLS für `api.mb-portal.com` vorbereiten; Health-Check prüfen:
   `curl https://<neue-ip>/auth/v1/health` bzw. über hosts-Eintrag.
5. Testlauf: auf dem eigenen Rechner per hosts-Datei `api.mb-portal.com` → neue IP
   setzen und Portal-Funktionen prüfen (Login, Chat, E-Mail-Versand-Log).

## Phase 3 — Frontend-Server neu aufbauen (parallel)

1. `bash scripts/setup-server2.sh` auf dem neuen Portal-Server (nginx/systemd).
2. Repo klonen, `.env` und `.env.server` vom Backup zurückspielen.
3. Erster Build + `systemctl restart portal`; Health-Check auf `http://localhost:3000`.
4. SSH-Zugänge der Nachbarserver einrichten: Deploy-Key, `ssh-copy-id` Richtung
   Landing-Server (.234) für den Landing-Sync.

## Phase 4 — Umschalten (das einzige Wartungsfenster)

1. Alte Portal-App anhalten (keine neuen Schreibvorgänge mehr verteilen).
2. Finaler Datenbank-Dump auf .123 und Restore auf den neuen Backend-Server.
3. Edge-Functions erneut deployen: `bash scripts/deploy-backend.sh`.
4. In Cloudflare die A-Records umstellen:
   - `mb-portal.com` (+ `www`) → neue Portal-IP
   - `api.mb-portal.com` → neue Backend-IP
5. Verifizieren (Checkliste):
   - `curl -I https://mb-portal.com` → 200
   - `curl https://api.mb-portal.com/auth/v1/health` → ok
   - Login im Browser, ein echter Chat/„Zuletzt aktiv"-Eintrag, CSV-Export
   - Landing-Page-Domain liefert 200, WebID-Sim-Domain liefert 200
6. Alte Server erst nach 24–48 h abschalten (Rückfallmöglichkeit).

## Phase 5 — IP-Referenzen anpassen (nach dem Umschalten)

Diese Stellen verweisen hart auf die alten IPs und müssen aktualisiert werden:

- `scripts/sync-to-backend.sh` / `scripts/backend-server.env` (Backend-IP .123)
- `TARGET_DB_URL` in `/opt/apps/portal/.env` auf dem Portal-Server
- `scripts/backup-orchestrator.env` (DB_HOST, SERVER_PORTAL_HOST) auf dem Backup-Server
- `ssh-copy-id` von Landing-, Bot- und WebID-Server auf die neuen Server,
  falls dort Syncs/Heartbeats hinführen
- Crontabs prüfen (Auto-Assign-Cron etc. — laufen über die Domain, bleiben unverändert)

## Offene Punkte

- genaue Größe des aktuellen Datenbank-Dumps prüfen (bestimmt die Dauer des
  Umschalt-Fensters mit) — beim Start messen
- falls der Umzug auf einen anderen Provider geht: ausgehende Mail-/SMTP-IPs
  können neu sein → SPF/DKIM prüfen, sonst landen Mails im Spam
