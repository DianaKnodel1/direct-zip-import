# Danke-Seite scharfschalten (Facebook-Tracking)

## Befund (geprüft)

- `https://app24-gmbh.com/bewerben` läuft bereits — Screenshot bestätigt das.
- `https://app24-gmbh.com/danke` antwortet ebenfalls schon mit Status 200, die Seite existiert also.
- Was fehlt: Das Formular-Skript auf der Live-Seite ist noch die **alte Fassung**.
  Im ausgelieferten Skript kommt das Wort "danke" kein einziges Mal vor — nach dem
  Absenden öffnet es weiterhin nur das Pop-up, statt auf `/danke` weiterzuleiten.
  Facebook sieht deshalb keinen Seitenaufruf, den es als Abschluss zählen könnte.

Ursache: Beim letzten Deploy wurden die Kern-Dateien des Landing-Servers übertragen,
der Abgleich der Theme-Dateien (dazu gehört das Formular-Skript) wurde aber
übersprungen — in der Deploy-Ausgabe stand dazu die Warnung
"DB-Update für themes_resync_requested_at fehlgeschlagen / kein TARGET_DB_URL".
Der Landing-Server holt neue Theme-Dateien nur, wenn genau diese Markierung gesetzt ist.

## Was gemacht wird

1. **Beschriftung korrigieren:** Die grüne Schaltfläche nach dem Absenden heißt
   künftig **„Jetzt Termin vereinbaren →"** statt „Jetzt bewerben →" — auf allen
   Landing Pages. Auch der begleitende Text wird auf die Terminauswahl bezogen.
2. **Theme-Abgleich anstoßen** — entweder über die Infrastruktur-Seite im Portal
   (Landing-Server → Themes neu synchronisieren) oder direkt auf dem Landing-Server.
   Danach lädt der Server Formular-Skript, Vorlagen und Bilder neu und startet sich neu.
3. **Zwischenspeicher leeren** und Dienste neu starten, damit die alten Seiten
   nicht weiter ausgeliefert werden.
4. **Prüfen:** Testbewerbung über `/bewerben` abschicken → die Adresse muss auf
   `/danke` wechseln, dort erscheint der Danke-Text und die Terminauswahl mit der
   neuen Beschriftung.
5. **Dauerhaft absichern:** `TARGET_DB_URL` in `/opt/apps/portal/.env.server`
   eintragen, damit künftige Deploys den Theme-Abgleich automatisch auslösen
   und dieser Zwischenschritt entfällt.

## Technische Details

- `landing-server/heartbeat.sh` synchronisiert Themes ausschließlich, wenn das
  Portal im Heartbeat `resync_needed: true` meldet; das hängt an der Spalte
  `themes_resync_requested_at` in `public.landing_servers`.
- `scripts/sync-landing-server.sh` setzt diese Spalte nur, wenn `TARGET_DB_URL`
  gesetzt ist — sonst nur eine Warnung, wie beim letzten Deploy.
- Die Danke-Weiterleitung selbst steckt in
  `src/landing-themes/_shared/form-section.js` (`location.assign('/danke…')`)
  und wird in die Theme-Datei `script.js` gebaut — deshalb ist der Theme-Abgleich
  der entscheidende Schritt.
- Am Code muss dafür nichts geändert werden; `/danke` und `/bewerben` sind im
  Renderer bereits vorhanden und live erreichbar.

## Danach für Facebook

Sobald die Weiterleitung greift, kann in den Meta-Anzeigen `/danke` als
Abschluss-Seite (Lead) genutzt werden. Ist im Landing-Generator eine
Pixel-ID hinterlegt, meldet die Danke-Seite das Ereignis "Lead" automatisch.
