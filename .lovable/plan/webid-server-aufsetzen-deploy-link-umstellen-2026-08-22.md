# WebID-Server aufsetzen + Deploy-Link umstellen

## Überblick

Drei Arbeitspakete in dieser Reihenfolge:
1. Deploy-Link im Repo auf `exactly-as-zipped.git` umstellen und deployen (dadurch laufen auch alle ausstehenden DB-Migrationen automatisch)
2. WebID-Server per PuTTY einrichten (fertige Befehle zum Abtippen)
3. Domain `webid-portal.de` kaufen, Wildcard-DNS setzen, Sim-Domain im Portal anlegen

Der Bot-Feinschliff ist bewusst **nicht** Teil dieses Plans — eigenes Vorhaben danach.

---

## 1. Deploy-Link umstellen (einzige Code-Änderung)

**Problem:** `scripts/deploy.sh` und `scripts/setup-server2.sh` zeigen noch auf das alte Repo `seamless-import-tool.git`.

**Änderung im Code (mache ich):**
- `scripts/deploy.sh` Zeile 13: `REPO_URL` → `https://github.com/DianaKnodel1/exactly-as-zipped.git`
- `scripts/setup-server2.sh` Zeile 19: `REPO_URL` → `https://github.com/DianaKnodel1/exactly-as-zipped.git`

**Einmalig auf dem Portal-Server (PuTTY), weil der Server sonst weiter das alte Repo zieht:**

```bash
cd /opt/apps/portal
git remote set-url origin https://github.com/DianaKnodel1/exactly-as-zipped.git
bash scripts/deploy.sh
```

Danach zieht jeder Deploy automatisch von `exactly-as-zipped`. Der Deploy spielt dabei auch alle offenen Migrationen ein (u. a. WebID-Tabellen, Chat-Fixes, Domain-Monitoring) — das erledigt den Punkt „Backend-Dateien deployen" von allein.

---

## 2. WebID-Server einrichten (PuTTY, Befehl für Befehl)

Voraussetzung: Schritt 1 ist durchgelaufen (sonst fehlt die Tabelle `webid_sim_domains` in der Datenbank).

**Auf dem neuen, nackigen Ubuntu/Debian-Server als root:**

```bash
# 1) Repo holen (nur der Ordner webid-sim-server wird gebraucht)
apt-get update && apt-get install -y git
git clone https://github.com/DianaKnodel1/exactly-as-zipped.git /tmp/portal
cd /tmp/portal/webid-sim-server

# 2) Setup-Skript ausführen (installiert Bun, Caddy, systemd-Service, .env)
SUPABASE_URL=https://iiabvudipyliimxjdpue.supabase.co \
SUPABASE_PUBLISHABLE_KEY=sb_publishable_XspVm9PLYoWcxyskSg1U-Q_Xcv5fe0d \
ACME_EMAIL=deine-email@mb-portal.com \
bash setup.sh
```

Das Skript installiert alles und startet am Ende den Dienst. Der Publishable-Key ist der öffentliche Schlüssel (kein Geheimnis) — der Server kann damit nur lesen, welche Sim-Domains aktiv sind.

**3) Prüfen, ob alles läuft:**

```bash
curl http://127.0.0.1:3002/_health        # muss "ok" antworten
systemctl status webid-sim caddy --no-pager
```

Wichtig: Ports **80 und 443** müssen von außen erreichbar sein (Firewall/Provider-Freigabe), sonst kann Caddy keine Zertifikate holen.

---

## 3. Domain kaufen + DNS + Sim-Domain anlegen

**3a) Domain kaufen:** `webid-portal.de` bei einem normalen Registrar kaufen (z. B. Cloudflare, IONOS, Namecheap). **Nicht** über Lovable kaufen — die Domain gehört nicht zur Lovable-App, sondern zu deinem eigenen Server.

**3b) DNS beim Registrar setzen** (Wildcard = alle Subdomains mit einem Eintrag):

| Typ | Name | Wert |
|---|---|---|
| A | `*` | IP des WebID-Servers |
| A | `@` | IP des WebID-Servers |

Falls der Registrar Cloudflare ist: Proxy-Schalter auf **„DNS only" (graue Wolke)** stellen — Caddy auf dem Server macht HTTPS selbst.

**3c) Sim-Domain im Portal anlegen:**
- Portal → `/admin/webid-sim` → Domain `uwk-consulting.webid-portal.de` eintragen, Anzeigename z. B. „UWK Consulting"
- Beim ersten Aufruf der Subdomain fragt Caddy automatisch beim Proxy nach („Ist die registriert?") → ja → Zertifikat wird automatisch ausgestellt. Das dauert beim ersten Aufruf wenige Sekunden.

**3d) Testen:** `https://uwk-consulting.webid-portal.de` im Browser öffnen → die gespiegelte WebID-Seite mit gelbem Simulations-Banner muss erscheinen.

---

## 4. Wie der Mitarbeiter die Sim-Domain bekommt

So läuft die Spiegelung (Antwort auf deine Frage):

```text
Mitarbeiter klickt Link im Auftrag
  → https://uwk-consulting.webid-portal.de
  → Caddy (Zertifikat) → WebID-Proxy (dein Server)
  → Proxy holt die Seite live von webid-gateway.de
  → baut Simulations-Banner + Blockade ein
  → Mitarbeiter sieht die echte WebID-Oberfläche als Simulation
```

Der Mitarbeiter macht **kein echtes Ident** — alle absendenden Aktionen (POST) werden mit „Simulation beendet" blockiert.

**Damit der Mitarbeiter den richtigen Link bekommt:**
- Im Auftrag (Admin-Ansicht) die **WebID-Start-URL** auf `https://uwk-consulting.webid-portal.de` setzen
- Der Mitarbeiter sieht den Link dann in seiner Auftragsansicht und startet die „Identifikation" darüber
- Pro Firma/Kampagne legst du einfach weitere Subdomains in `/admin/webid-sim` an (z. B. `firma-b.webid-portal.de`) — ohne neue Domain zu kaufen, dank Wildcard-DNS

---

## Technische Details

- **Code-Änderungen:** nur `scripts/deploy.sh` + `scripts/setup-server2.sh` (je 1 Zeile, REPO_URL)
- **Keine neuen Migrationen nötig** — `20260731000000_webid_sim_domains.sql` und `20260810000000_tenant_webid_enabled.sql` existieren bereits und werden vom Deploy automatisch eingespielt
- **WebID-Server-Architektur:** Bun-Proxy auf 127.0.0.1:3002, Caddy davor mit On-Demand-TLS; Domain-Freigabe wird alle 60 s gecacht aus `webid_sim_domains` gelesen (anon key, nur lesend)
- **Sicherheit:** Rate-Limit 120 req/min, POST blockiert (außer `allow_submit`), robots.txt sperrt Suchmaschinen aus, Overlay kennzeichnet alles als Simulation
- **Nach Deploy verifizieren:** Domain-Monitoring-Banner im Admin-Dashboard und WebID-Schalter pro Tenant prüfen

## Nicht in diesem Plan

- Bot-Feinschliff (Deutsche-Bank-Strecke etc.) — eigenes Vorhaben, sobald Setup steht
- Weitere Sim-Domains/Logos pro Kampagne — kannst du nach dem Setup jederzeit selbst im Portal anlegen
