# Remote-Umstellung auf `direct-zip-import` + Deploy

Neues Remote: `https://github.com/DianaKnodel1/direct-zip-import.git` (Branch `main`, Repo public angenommen — falls Token nötig, bitte vorher melden).

Ziel-Server: **Frontend**, **WebID (servervps)**, **Bot**.
Nicht angefasst: backendserver, landing-page-server (werden über `deploy.sh` vom Frontend mitgezogen).

## Repo-Pfade (bestätigt)

- Frontend: `/opt/apps/portal` (aktuell `exactly-as-zipped.git`)
- WebID (servervps): `/opt/apps/portal` (aktuell `exactly-as-zipped.git`)
- Bot: Pfad noch offen — Discovery-Schritt unten

Alte/Backup-Repos auf Frontend (`portal.wrong-repo`, `portal.bak-*`) werden nicht angefasst.

## Ablauf pro Server

Für jeden Server derselbe Ablauf:

1. Backup des aktuellen Working Trees als Sicherheitsnetz (`portal.bak-<timestamp>`).
2. Remote auf neues Repo umbiegen (`git remote set-url origin …`).
3. `git fetch origin` + `git reset --hard origin/main` (sauberer Stand aus dem neuen Repo).
4. Deploy triggern.

## Deploy-Kommandos pro Server

- Frontend `/opt/apps/portal`: `bash deploy.sh` (zieht landing-page + backend-Sync mit).
- WebID `/opt/apps/portal` auf servervps: `bash setup.sh` bzw. `systemctl restart webid-sim` nach `bun install` + Build (genaues Skript wird aus dem Repo-Root gelesen, kein manuelles Raten).
- Bot: Deploy-Skript aus dem Repo-Root (üblicherweise `deploy.sh` oder `start.sh`) — konkreter Befehl steht nach Discovery fest.

## Discovery-Schritt für den Bot-Server (einmalig)

Auf dem Bot-Server ausführen, damit Pfad + aktuelles Remote sichtbar werden:

```bash
find /opt /root /home -maxdepth 5 -type d -name ".git" 2>/dev/null | while read g; do
  d=$(dirname "$g"); echo "=== $d ==="; git -C "$d" remote -v
done
```

Sobald Pfad bekannt: gleicher 4-Schritt-Ablauf wie oben.

## Rollback

Falls nach dem Deploy etwas kaputt ist: `systemctl stop <service>` → Backup-Ordner zurückverschieben → Service neu starten. Kein DB-Eingriff nötig, es werden nur App-Dateien getauscht.

## Offene Punkte vor Ausführung

1. Ist das neue Repo public, oder braucht es einen Token / Deploy-Key? Falls Token: liegt der schon auf den Servern (`~/.git-credentials` / SSH-Key)?
2. Pfad des Bot-Servers (Output des Discovery-Kommandos).
