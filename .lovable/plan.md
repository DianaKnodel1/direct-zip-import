# WebID-Sim vereinfachen und Fehler eingrenzen

Ziel: Ohne Rätselraten sehen, **welcher Request** hängt, **wohin** er geht und **was** die Antwort ist. Danach den kleinsten möglichen Fix.

## Was wir vermuten (unbestätigt)

Der Proxy schreibt HTML/CSS nur für **eine** `target_origin` (z. B. `webid-gateway.de`) um. Das WebID-Widget spricht aber sehr wahrscheinlich zusätzlich mit **anderen Hosts** (`service.webid-solutions.de`, `webid-solutions.de` o. ä.). Requests dorthin gehen entweder direkt an die echte Origin (dann fehlen Cookies/Session) oder werden relativ an unsere Sim-Domain gesendet und laufen bei uns ins Leere. Nichts davon ist verifiziert — deshalb erst messen.

Zusätzlich haben wir einen `MutationObserver` im Overlay, der bei DOM-Änderungen `location.reload()` triggert (server.ts Z. 158-163). Das kann die Widget-Initialisierung im Kreis reloaden.

## Schritt 1 – Diagnose (kein Code-Change)

Zwei simple Messungen, mehr nicht:

1. **HAR-Datei** vom „Überprüfen"-Klick.
   - DevTools → Network → „Preserve log" an → „Fetch/XHR" Filter → auf „Überprüfen" klicken → Rechtsklick in Liste → **„Save all as HAR with content"** → Datei schicken.
2. **Server-Log parallel dazu**:
   ```bash
   journalctl -u webid-sim -n 500 --no-pager | tail -200
   ```

Aus der HAR lesen wir ab: Zielhost, Status, Response-Body des ersten Fehlers. Damit steht der Fix fest.

## Schritt 2 – Vereinfachung des Proxys (nach Diagnose, klein halten)

Nur die drei kleinsten Vereinfachungen, die den Fehler sichtbar machen und typische Ursachen ausschließen:

**A. Reload-Schleife entschärfen** (`webid-sim-server/server.ts`, Z. 157-163)
Statt `location.reload()` bei fehlenden Overlay-Elementen: Elemente einfach neu einfügen. Verhindert, dass unser Watchdog die Widget-Initialisierung tötet.

**B. Server-seitiges Request-Log** hinzufügen (eine Zeile pro Request):
`[webid-sim] METHOD host path → upstream status`
Damit sieht `journalctl` sofort, welcher Request wo landet.

**C. Alle „unbekannten" WebID-Hosts explizit ablehnen mit 501 + Log** (statt still 404):
Wenn das Widget einen fremden Host aufruft und der über unsere Sim-Domain kommt, schreibt der Proxy `[MISSING UPSTREAM] host=… path=…` in den Log. So wird ohne HAR sichtbar, welche zusätzlichen Hosts wir brauchen.

Danach entscheiden wir anhand des Logs, ob wir Multi-Upstream-Routing brauchen (neue Spalte `extra_hosts` in `webid_sim_domains` + Rewrite über `/__u/<host>/…`) oder ob eine kleinere Änderung reicht.

## Was wir bewusst NICHT jetzt machen

- Kein Multi-Upstream-Umbau blind.
- Kein Cookie-/Domain-Rewrite-Overhaul.
- Keine DB-Migration, bevor der Log zeigt, dass wir sie brauchen.

## Deliverable Schritt 1

Du schickst mir HAR + Logauszug. Ich sage dir dann konkret: „Es fehlt Host X, wir brauchen Fix Y" — und baue nur den.
