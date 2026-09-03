# WebID-Fehler ohne HAR finden

## Einfach erklärt

Die Adresse im Browser ist nur die **erste Seite**:

```text
Browser → bv-agentur.webid-portal.com → Proxy → WebID-Startseite
```

Dass diese Seite erscheint, beweist nur: **Der erste GET-Aufruf funktioniert.**

Nach dem Klick auf „Weiter/Überprüfen“ startet das WebID-JavaScript weitere, unsichtbare Anfragen, zum Beispiel:

```text
/api/web-product/...
/cgate/self-ident-status/...
oder Anfragen an einen weiteren WebID-Host
```

Unser Proxy baut derzeit jeden Pfad gegen genau eine konfigurierte `target_origin`. Wenn `/service/precheck/...` und die späteren API-Pfade tatsächlich zu unterschiedlichen WebID-Hosts gehören, schickt er mindestens einen Aufruf an den falschen Server. Deshalb kann die sichtbare Seite funktionieren, während das Widget bei „Initialisiere…“ stehen bleibt.

Wichtig: Der Wechsel von `/service/index/...` zu `/service/precheck/...` ist vermutlich ein normaler Redirect innerhalb des WebID-Ablaufs und nicht automatisch der Fehler.

## Vereinfachter Diagnoseweg

Wir verzichten auf HAR und DevTools-Auswertung.

### 1. Proxy protokolliert jeden Aufruf verständlich

In `webid-sim-server/server.ts` ergänzen wir pro Anfrage eine kompakte Zeile:

```text
[webid-sim] POST /cgate/... → https://webid-gateway.de/cgate/... → 502
```

Bei Fehlerantworten protokollieren wir zusätzlich einen kurzen, begrenzten Ausschnitt der Upstream-Antwort. Cookies, Tokens und Formulardaten werden nicht geloggt.

### 2. Mögliche Reload-Schleife entfernen

Der vorhandene Overlay-Watchdog führt aktuell `location.reload()` aus, sobald WebID eines unserer Overlay-Elemente entfernt. Das kann die Initialisierung selbst unterbrechen. Statt neu zu laden, setzt er nur das fehlende Simulationselement wieder ein.

### 3. Danach nur einmal testen

Nach Deployment:

1. Simulationsseite öffnen.
2. Einmal auf „Weiter/Überprüfen“ klicken und warten, bis „Initialisiere…“ erscheint.
3. Auf dem Portal-Server ausführen:

```bash
journalctl -u webid-sim --since "5 minutes ago" --no-pager
```

Die Ausgabe zeigt dann direkt den fehlerhaften Pfad, Zielserver und Status. Kein HAR und kein Suchen in Browser-Menüs nötig.

## Anschließender Fix

Erst anhand dieses Logs wird der Routing-Fix festgelegt:

- **Falscher Zielhost:** gezieltes Routing pro WebID-Pfad bzw. erlaubtem WebID-Host ergänzen.
- **OPTIONS/CORS fehlerhaft:** Preflight und CORS-Header vollständig an die Sim-Domain anpassen.
- **Cookie/Session fehlt:** Cookie-Domain und Weitergabe für den betroffenen Host korrigieren.
- **Reload im Log/Browser:** Overlay-Watchdog war die Ursache; die Änderung aus Schritt 2 behebt sie.

So ändern wir nicht blind das gesamte Proxy-System, sondern brauchen nach dem Deployment nur noch einen Klick und einen einfachen Log-Befehl.
