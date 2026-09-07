# Neuer Landing-Page-Server: Setup abschließen

## Aktueller Stand
- Server gekauft, SSH-Login funktioniert, Code liegt unter `/opt/src/portal/landing-server`.
- `setup.sh` ist am Export-Schritt gescheitert: Die Platzhalter `<anon-key>` und `<Token aus /admin/infrastructure>` wurden mit den spitzen Klammern eingetippt — die Shell bricht bei `<`/`>` ab. Deshalb fehlte `SUPABASE_PUBLISHABLE_KEY`.

## Nächste Schritte (vom User auf dem Server auszuführen)

### 1. Echte Werte besorgen
- **Anon-Key** (`SUPABASE_PUBLISHABLE_KEY`): im Portal unter `/admin/infrastructure` bzw. aus der Backend-Konfiguration (derselbe Key wie im Frontend `.env`).
- **Bootstrap-Token** (`LANDING_SERVER_TOKEN`): im Portal unter `/admin/infrastructure` erzeugen/anzeigen.

### 2. Exports erneut setzen — mit echten Werten, OHNE spitze Klammern
```bash
export SUPABASE_URL=https://api.mb-portal.com
export SUPABASE_PUBLISHABLE_KEY=hier-den-echten-key-einfuegen
export PORTAL_API_ENDPOINT=https://portal.mb-portal.com/api/public/applications
export ACME_EMAIL=admin@mb-portal.com
export LANDING_SERVER_TOKEN=hier-den-echten-token-einfuegen
bash setup.sh
```
Wichtig: Werte direkt einfügen, keine `<` `>`, keine Anführungszeichen nötig (außer der Wert enthält Leerzeichen — tut er normalerweise nicht).

### 3. Verifikation nach dem Setup
```bash
systemctl status landing caddy landing-agent --no-pager | head -30
curl -s http://127.0.0.1:3000/health
```

### 4. DNS umstellen
Alle Landing-Domains in Cloudflare per A-Record auf die IP des neuen Servers zeigen lassen; Caddy holt die Zertifikate dann automatisch beim ersten Aufruf.

### 5. Alten Landing-Server außer Betrieb nehmen
Erst wenn neue Domains mit HTTPS erreichbar sind, alten Server (.234) abschalten/kündigen.

## Technische Details
- Kein Code-Eingriff nötig; es ist ein reiner Shell-Bedienungsfehler gewesen.
- Falls `setup.sh` erneut meckert, welche Variable fehlt, die Ausgabe hier einfügen.
