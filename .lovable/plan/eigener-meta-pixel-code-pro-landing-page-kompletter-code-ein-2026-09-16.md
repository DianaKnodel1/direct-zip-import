# Eigener Meta-Pixel-Code pro Landing Page (kompletter Code einfügbar)

## Ziel

Im Landing-Generator kannst du künftig den **kompletten Pixel-Code** deines Sellers 1:1 reinkopieren — nicht nur die ID. So kannst du den Code jederzeit selbst austauschen, wenn Meta bzw. dein Seller eine neue Fassung schickt.

## Befund (geprüft)

- Bereits vorhanden und live: Feld **Facebook-Pixel-ID** (`meta_pixel_id`) im Landing-Generator unter Branding. Der Landing-Server baut daraus den Standard-Pixel-Code (init + PageView auf allen Seiten, Lead auf `/danke`, Consent-Gate in DE/EU).
- Der Code deines Sellers unterscheidet sich nur in der ID (`2113512246264776`) — technisch würde also das bestehende ID-Feld reichen. Damit du aber nie prüfen musst, ob sich am Code selbst etwas geändert hat, kommt ein Feld für den kompletten Code dazu.

## Umsetzung

1. **Neues Feld `meta_pixel_code`** (kompletter Code als Text) pro Landing Page:
   - `src/lib/landing-pages.functions.ts`: BrandingSchema um `meta_pixel_code` (Text, max. 10.000 Zeichen, optional) erweitern.
   - `src/routes/admin.landing-generator.tsx`: Mehrzeiliges Textfeld „Eigener Meta-Pixel-Code (komplett)" unter dem ID-Feld, mit Hinweis: *„Kompletten Code vom Werbeberater hier reinkopieren — überschreibt die Pixel-ID oben."*
2. **Landing-Server** (`landing-server/server.ts` + `server.js`, parallel pflegen):
   - `buildPixelBlock`: Wenn `meta_pixel_code` gesetzt ist, diesen Code unverändert in `</head>` einfügen (statt des aus der ID gebauten Blocks).
   - Lead-Event auf `/danke` bleibt erhalten: Nach dem eingefügten Code feuert weiterhin automatisch `fbq('track', 'Lead')` (das eingefügte Skript stellt `fbq` bereit).
   - Consent-Gate (`lv_ads_consent`) gilt weiterhin: Code wird erst nach „Einverstanden" geladen.
   - Sicherheit: Feld ist Admin-only (Landing-Generator erfordert Admin-Rolle); Code wird nur in den eigenen Landing-Kontext eingefügt.
3. **Vorrang-Logik:** `meta_pixel_code` gefüllt → eigener Code. Sonst `meta_pixel_id` gefüllt → Standard-Code. Beides leer → kein Pixel.

## Prüfung

- Build + `node --check landing-server/server.js`.
- Nach Deploy: Bei einer Test-Landing den Seller-Code einfügen, Seite öffnen, Consent klicken → Netzwerk-Anfrage an `facebook.com/tr` mit `id=2113512246264776` und `ev=PageView`; Testbewerbung → `ev=Lead` auf `/danke`.

## Veröffentlichung

Codeänderung am Landing-Server → danach Deploy nötig (`git pull && bash scripts/deploy.sh` auf dem Portal-Server), damit die neue `server.js` auf den Landing-Server übertragen wird. Ein Theme-Neuabgleich ist nicht erforderlich.
