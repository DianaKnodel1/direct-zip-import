# Meta-Pixel für Facebook-Ads scharfschalten

## Befund (geprüft)

Die Pixel-Unterstützung ist im Landing-Server **bereits vollständig eingebaut und live** (mit dem letzten Deploy ausgerollt) — es muss kein Code mehr geändert werden:

- **Feld vorhanden:** Im Landing-Generator gibt es unter „Branding" das Feld **Facebook-Pixel-ID** (`meta_pixel_id`). Es liegt pro Landing Page im Branding — dadurch funktioniert es automatisch auf **jeder Domain**, nicht nur app24-gmbh.com. Kein hartcodierter Domain-Bezug.
- **Basis-Code:** Der Landing-Server baut bei jedem Seitenaufruf den Meta-Pixel-Code in `</head>` ein (`landing-server/server.ts`/`server.js`, `buildPixelBlock`) — auf allen Seiten: Startseite, `/bewerben`, `/danke`. Es wird `fbq('init', …)` + `fbq('track', 'PageView')` gefeuert, identisch zum eingefügten Standard-Snippet.
- **Lead-Ereignis:** Auf `/danke` feuert das Pixel automatisch zusätzlich `fbq('track', 'Lead')`. Da `/danke` ausschließlich nach erfolgreich abgeschicktem Formular erreicht wird, ist das exakt der geforderte Zeitpunkt („Fast geschafft!"-Moment).
- **Einwilligung (DSGVO):** In einwilligungspflichtigen Regionen (DE/EU) erscheint ein schlanker Hinweis unten; das Pixel lädt erst nach „Einverstanden", Ablehnen ist gleichwertig möglich, jederzeit änderbar. Außerhalb dieser Regionen lädt das Pixel direkt. Die Datenschutzerklärung der Landings enthält Meta-Pixel, Zweck und Widerruf bereits (Version 2026-09-15.1).
- **Wirkt ohne Deploy:** Der Landing-Server liest die Landing-Daten pro Seitenaufruf (60 Sekunden Zwischenspeicher). Eintrag speichern genügt — kein Deploy, kein Theme-Abgleich.

## Was zu tun ist

1. **Pixel-ID eintragen:** Im Portal → Landing-Generator → Landing Page von app24-gmbh.com öffnen → Branding → **Facebook-Pixel-ID = `1778455953439812`** → speichern.
2. **Weitere Domains:** Für jede weitere Landing Page (eigene Domain) dort die Pixel-ID des jeweiligen Meta-Werbekontos eintragen — das System ist bewusst pro Seite, damit nie ein fremdes Pixel auf einer fremden Domain feuert.
3. **Prüfen (Browser):** `https://app24-gmbh.com` öffnen, im Consent-Hinweis „Einverstanden" klicken, in den Entwicklerwerkzeugen (Netzwerk) prüfen, dass eine Anfrage an `facebook.com/tr` mit `ev=PageView` geht. Danach Testbewerbung über `/bewerben` abschicken → auf `/danke` muss eine Anfrage mit `ev=Lead` erscheinen.
4. **Meta-Seite:** Im Meta Events Manager kontrollieren, dass PageView und Lead ankommen. Danach kann in den Kampagnen „Lead" als Conversion-Ziel bzw. `/danke` als Abschluss-URL genutzt werden.

## Kein Codeaufwand, keine Risiken

- Ohne Pixel-ID im Feld wird nichts geladen und kein Hinweis angezeigt — bestehende Landings bleiben unverändert.
- Der Consent-Hinweis erscheint nur, wenn eine Pixel-ID hinterlegt ist.
