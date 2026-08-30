# Landing-Server automatisch beim Portal-Deploy syncen

## Ziel
Wenn `scripts/deploy.sh` auf dem Portal-Server (.124) läuft, soll danach auch der Landing-Server (`uwkconsulting`) auf den neuen Stand gebracht werden — ohne manuelles `rsync` oder Warten auf den Heartbeat.

## Ist-Zustand
- `scripts/deploy.sh` baut das Portal und spielt Migrationen auf dem Backend (.123) ein, synchronisiert aber **nicht** den Landing-Server.
- Der Landing-Server liegt auf `uwkconsulting` unter `/opt/landing-server` und ist kein Git-Repo.
- Es gibt bereits einen Pull-Mechanismus (`/api/public/landing-server-files/$` + `landing-server/heartbeat.sh`), der auf `themes_resync_requested_at` reagiert.
- Der Heartbeat läuft nur alle ~60 s; ein sofortiger, deterministischer Sync beim Deploy fehlt.

## Geplante Änderungen

### 1. Neues Sync-Skript: `scripts/sync-landing-server.sh`
- Kopiert lokal gebaute Landing-Server-Dateien auf den Zielserver:
  - `landing-server/server.js`
  - `landing-server/legal-content.js`
  - `landing-server/package.json`
  - `landing-server/heartbeat.sh`
  - generierte Themes unter `landing-server/themes/` (oder direkt aus `src/landing-themes` + `theme-assets.generated.ts`)
- Zielhost und Pfad konfigurierbar über `LANDING_SYNC_HOST` (z. B. `root@uwkconsulting`) und `LANDING_SYNC_PATH` (Default `/opt/landing-server`).
- Führt auf dem Zielserver aus:
  - `systemctl restart landing-server.service`
  - `systemctl restart landing-heartbeat.service` (falls vorhanden)
  - Health-Check `curl http://127.0.0.1:3001/_health`
  - Prüfung `window.PORTAL_API` via `curl https://tb-app.de/`
- Bricht bei Fehlern ab und gibt lesbare Logs aus.

### 2. Integration in `scripts/deploy.sh`
- Nach erfolgreichem Portal-Build und -Restart wird `scripts/sync-landing-server.sh` aufgerufen, wenn `LANDING_SYNC_HOST` gesetzt ist.
- Der Schritt ist idempotent und optional abschaltbar (z. B. `SKIP_LANDING_SYNC=1`).

### 3. Fallback/Alternative: Pull-Trigger über Heartbeat
- `src/routes/api/public/landing-server-heartbeat.ts` setzt `resync_needed:true`, wenn `themes_resync_requested_at` neuer ist als der letzte bekannte Sync-Zeitstempel des Agents.
- `landing-server/heartbeat.sh` wird robuster gemacht:
  - Prüft Downloads auf HTML-Fehlerseiten.
  - Synced auch `server.js` + `legal-content.js` zuverlässig.
  - Loggt Erfolg/Fehler deutlich.

### 4. Admin-UI: Manuelles Syncen
- `src/routes/admin.infrastructure.tsx` erhält einen zusätzlichen Button „Landing-Server jetzt syncen".
- Der Button setzt `themes_resync_requested_at = now()` für den registrierten Server und/oder führt `scripts/sync-landing-server.sh` serverseitig aus.
- Zeigt Erfolg/Fehler als Toast.

## Technische Details
- SSH-Key-basierte Authentifizierung vorausgesetzt (keine Passwörter im Skript).
- Das Sync-Skript nutzt `rsync -avz --delete` für Themes, damit alte Theme-Dateien entfernt werden.
- `server.js` und `legal-content.js` werden nur gemeinsam ausgetauscht, damit keine Versionen gemischt werden.
- Keine Breaking Changes: Bestehende Heartbeat-Logik bleibt erhalten.

## Offene Entscheidung
- Soll der Push-Sync nur `uwkconsulting` hartkodieren oder über `landing_servers`-Tabelle dynamisch ermittelt werden? → Für den Plan wird `uwkconsulting` als einziger bekannter Server konfiguriert; dynamische Erweiterung kann später folgen.
