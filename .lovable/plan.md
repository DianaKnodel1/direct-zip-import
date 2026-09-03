# Diagnose: WebID-Simulation bleibt bei „Initialisiere..." hängen

Nur Analyse — keine Änderungen. (Roadmap-Notiz folgt im Build-Mode.)

## Neue Beobachtung (Screenshots)

- URL ist auf Sim und Original **identisch**:
  `/service/precheck/cn/000631/aid/695856893`
- Der Proxy liefert HTML, CSS, Bilder korrekt (Layout, Vorgangsnummer,
  Schritt-1/Schritt-2-Text passen 1:1 zum Original).
- **Unterschied nur im blauen Widget unten**:
  - Original: „Halten Sie Ihren Personalausweis... **Online-Identifikation starten**"
  - Sim: nur „Initialisiere.." — Widget bleibt hängen, der Button erscheint
    nie, ein Klick auf „Weiter" oben kommt nie an.

Damit ist das URL-/Redirect-Thema aus dem vorigen Plan **nicht** das Problem
— die Strecke landet auf der richtigen Seite. Das Problem ist die
**Widget-Initialisierung im Browser**.

## Was das Widget vermutlich macht

Der WebID-Precheck-Bereich ist eine JS-Komponente, die beim Laden:

1. eine Konfiguration/Session per `fetch`/`XHR` von einer WebID-API zieht
   (typisch: `https://webid-gateway.de/api/…`, `https://service.webid-solutions.de/…`
   oder ein WebSocket wie `wss://…webid…/ws`),
2. eine kleine Vorabprüfung (Kamera-/Geräte-Fähigkeiten, Session-Cookies) macht,
3. erst dann den „Online-Identifikation starten"-Button rendert.

Kommt einer dieser Schritte nicht durch, bleibt die Anzeige auf „Initialisiere..".

## Wahrscheinliche Ursachen (nach Wahrscheinlichkeit)

1. **Cross-Origin-Requests des Widgets werden blockiert.**
   Unser Proxy schreibt HTML nur für **eine** `targetHost`
   (`row.target_origin`, `webid-sim-server/server.ts` → `rewriteHtml`, Z. ~190).
   Ruft die Widget-JS z. B. `service.webid-solutions.de` oder `webid-solutions.de`
   auf, geht der Request an die **echte** WebID-Origin — dort fehlt die Session,
   CORS blockt, Cookies (`SameSite=None`, Domain `.webid-portal.com`) sind
   nicht gesetzt → das Widget hängt. Passt zur Beobachtung, dass das
   restliche HTML sauber aussieht, nur die dynamische Sektion tot bleibt.

2. **Unser MutationObserver reloadet die Seite, sobald das Widget das DOM anfasst.**
   Im injizierten Overlay-Script steht:
   ```js
   var mo=new MutationObserver(function(){
     ['__webid_sim_topbar','__webid_sim_badge','__webid_sim_style'].forEach(function(id){
       if(!document.getElementById(id)){location.reload();}
     });
   });
   mo.observe(document.documentElement,{childList:true,subtree:true});
   ```
   Wenn das Widget beim Init Teile ersetzt (z. B. `document.write` oder ein
   Full-Body-Replace) und dabei unsere Elemente aus dem DOM fliegen,
   triggert der Reload eine Endlosschleife → Widget kommt nie über
   „Initialisiere.." hinaus. Das ist im Browser leicht am „loading"-Wheel
   im Tab bzw. an einer Reload-Kaskade im Network-Tab zu sehen.

3. **CSP/Cookie-Kombi verhindert die Widget-JS.**
   Wir setzen zwar eine sehr offene CSP, entfernen aber auch `HSTS`,
   `X-Frame-Options` etc. Falls die Widget-JS per `<script type=module>`
   oder `import()` von einer nicht gerewriteten Origin nachlädt und dabei
   `crossorigin`/`integrity` prüft, kann sie stillschweigend abbrechen.

4. **Kein `Referer` mit passendem Host.**
   Wir setzen zwar `Referer` neu (`Zeile ~250 im server.ts`), das gilt aber
   nur für den Initial-Request — die vom Browser danach ausgehenden
   XHR-Requests des Widgets tragen den Sim-Host als Referer. Manche WebID-
   Backends prüfen den Referer strikt und antworten dann mit 403/200-leer.

## Was wir vor jedem Fix messen sollten

Alles live im Browser auf `https://bv-agentur.webid-portal.com/service/precheck/cn/000631/aid/695856893`:

1. **DevTools → Netzwerk → „Preserve log"**, dann Seite neu laden.
   - Auffällig: alle Requests, die **nicht** an `bv-agentur.webid-portal.com`
     gehen (z. B. `webid-gateway.de`, `*.webid-solutions.de`, `*.webid.de`,
     `wss://…`) → das ist die Liste der Origins, die die Sim mit-spiegeln muss.
   - Status/CORS-Fehler notieren (rote Zeilen, `blocked:cors`, `blocked:mixed-content`).

2. **DevTools → Konsole** — nach roten Fehlern suchen
   („Refused to connect", „blocked by CORS", „Failed to fetch",
   „Uncaught (in promise)"). Das zeigt sofort, ob es Ursache 1 oder 3 ist.

3. **Tab-Titel/URL beobachten**: springt die Adressleiste im Sekundentakt
   um oder wird die Seite ständig neu geladen? → dann ist unser
   MutationObserver-Reload (Ursache 2) im Spiel.

4. **Auf dem WebID-Server** parallel:
   `journalctl -u webid-sim -n 200 --no-pager | grep -iE 'upstream|502|blocked'`
   — sehen wir dort einen 502/Upstream-Error genau beim „Weiter"-Klick, ist
   ein Fremd-Host ohne Proxy im Spiel.

## Ausblick (Fix-Skizze, nur zur Info, noch nicht bauen)

Je nachdem, was die Messwerte zeigen, sind das die naheliegenden Fixe:

- **Ursache 1/3**: `webid_sim_domains` bekommt eine Liste **erlaubter
  Upstream-Hosts**. Der Proxy schreibt HTML/JS/CSS/`Location`/`Set-Cookie`
  für alle diese Hosts auf die Sim-Domain um und routet Requests via
  Präfix (`/__u/<host>/…`) an den passenden Upstream. Erst dann laufen
  XHR/WebSocket der Widget-JS innerhalb der Sim-Domain.
- **Ursache 2**: MutationObserver so umbauen, dass er unsere Elemente
  **wieder einsetzt** statt `location.reload()` zu triggern.
- Zusätzlich: JS-Response-Rewrite (bisher rewriten wir nur HTML + CSS,
  nicht `application/javascript`) — dort stehen die Fremd-Host-URLs oft
  hartkodiert drin.

## Technische Fundstellen

- Proxy: `webid-sim-server/server.ts`
  - HTML-Rewrite: `rewriteHtml` (~Z. 190) — nur ein `targetHost`
  - JS-Responses werden **nicht** umgeschrieben (nur HTML/CSS)
  - MutationObserver-Reload im Overlay-Script (~Z. 170)
  - `Set-Cookie`-Rewrite nur für `targetHost`: `copySetCookie` (~Z. 305)
- Domain-Konfig: `public.webid_sim_domains.target_origin`
  (Migration `supabase/manual-migrations/20260731000000_webid_sim_domains.sql`).

Bis die vier Messpunkte oben da sind, macht ein Code-Fix keinen Sinn —
sonst raten wir am eigentlichen Blocker vorbei.
