# Migrations-Ausgabe fehlt im Deploy-Log

## Beobachtung

Im letzten Deploy-Log auf dem Frontend-Server bricht die Ausgabe direkt nach dem Vite-Build ab (`✔ You can preview this build...` / `▶ Restart… ▶ Healthcheck…`). Es fehlt komplett:

- `▸ 3/5  release activation`
- `▸ 4/5  migrations` inkl. „Applying …" pro `.sql`
- `▸ 5/5  restart`
- `✓ Deploy finished`

Das sind exakt die Stufen aus `scripts/deploy.sh` (das Skript, das früher die Migrations-Ausgabe erzeugt hat).

## Wahrscheinliche Ursache

Auf dem Server wird gerade **nicht** `scripts/deploy.sh` ausgeführt, sondern ein anderes `deploy.sh` (z.B. `/opt/apps/portal/deploy.sh`). Hinweise:

- Im vorherigen Fehler kam `deploy.sh: line 17: ... bun run build` — `scripts/deploy.sh` hat in Zeile 17 aber `log()`-Definitionen, kein `bun run build`.
- Die Ausgabe endet mit `▶ Restart… ▶ Healthcheck…` — das sind Nitro/Wrangler-Post-Hooks, nicht unsere `log`-Zeilen.
- Damit läuft der ganze Block „4/5 migrations" nie an, und die `manual-migrations/*.sql` werden nicht gegen den Backend-DB gespielt → keine Ausgabe, kein State-Update in `.deploy-migrations-applied`.

Zusätzlich möglich, aber sekundär: selbst wenn `scripts/deploy.sh` liefe, würde der Migrations-Block still übersprungen, wenn
- `TARGET_DB_URL` in `.env`/`.env.server` fehlt, oder
- Port `5432` auf `190.97.167.123` vom Frontend-Server nicht erreichbar ist und `scripts/sync-to-backend.sh` fehlt.

## Plan

1. **Auf dem Frontend-Server prüfen, welches Skript wirklich läuft** und ob das erwartete Skript existiert:

   ```bash
   ls -la /opt/apps/portal/deploy.sh /opt/apps/portal/scripts/deploy.sh 2>/dev/null
   head -20 /opt/apps/portal/deploy.sh 2>/dev/null
   ```

   - Wenn `/opt/apps/portal/deploy.sh` ein eigenes Mini-Skript ist → das ist der Grund. Fix: dieses File durch einen Wrapper ersetzen, der `scripts/deploy.sh` aufruft (oder direkt `bash scripts/deploy.sh` statt `bash deploy.sh` verwenden).

2. **Migrations-Voraussetzungen verifizieren** (nur nötig, sobald wieder `scripts/deploy.sh` läuft):

   ```bash
   grep -E '^(TARGET_DB_URL|SUPABASE_URL)=' /opt/apps/portal/.env* 2>/dev/null
   nc -z -w 2 190.97.167.123 5432 && echo "DB reachable" || echo "DB NOT reachable"
   ls /opt/apps/portal/supabase/manual-migrations/*.sql | wc -l
   cat /opt/apps/portal/.deploy-migrations-applied 2>/dev/null | tail
   ```

3. **Deploy erneut ausführen** mit dem richtigen Skript:

   ```bash
   cd /opt/apps/portal && bash scripts/deploy.sh
   ```

   Ausgabe sollte wieder `▸ 4/5  migrations` inkl. `· Applying …` zeigen. Wenn nichts „Applied" wird, sind entweder alle SQLs bereits in `.deploy-migrations-applied` als angewendet markiert (dann ist das korrekt — nichts Neues), oder der `nc`-Check schlägt fehl und wir fallen auf `sync-to-backend.sh` zurück.

## Warum kein Code-Change nötig

Der Fix ist rein operativ (falsches Skript wird aufgerufen bzw. `TARGET_DB_URL`/Netzwerkzugang prüfen). Erst wenn Punkt 1 klar zeigt, dass `scripts/deploy.sh` tatsächlich läuft und die Migrations trotzdem nicht ausgegeben werden, ergänzen wir Logging (z.B. echo, wie viele `.sql` gefunden wurden, bevor die Schleife startet).
