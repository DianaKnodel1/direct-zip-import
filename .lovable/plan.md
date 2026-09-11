# Blog-Beiträge als eigene Seiten (Theme "Mirror Site")

## Problem

Auf bcu-beratung.com führen die drei Blog-Kacheln beim Klick zum Bewerbungsformular statt zu einem Beitrag. Im Theme sind die Kachel-Überschriften fest auf das Formular verlinkt.

## Lösung

Jeder der drei Beiträge bekommt eine eigene Unterseite, z. B. `/blog/1`, `/blog/2`, `/blog/3` — mit Bild, Datum, Titel, vollständigem Text, Zurück-Link und einem Bewerben-Button am Ende.

### Was gemacht wird

1. **Neue Textfelder pro Beitrag** im Landing-Page-Editor: zusätzlich zu Titel, Datum und Auszug ein Feld "Beitragstext" (mehrzeilig). Ohne eigenen Text wird der Auszug angezeigt, damit bestehende Seiten sofort funktionieren.
2. **Kacheln verlinken** auf die jeweilige Beitragsseite statt auf das Formular (Titel und Bild klickbar, plus "Weiterlesen").
3. **Beitragsseiten ausliefern**: Der Landing-Server beantwortet `/blog/1..3` mit einer eigenen Seite im Stil der Landing Page (gleiche Kopf-/Fußzeile wie die bestehenden Impressum-/Datenschutz-Seiten). Unbekannte Nummern leiten zurück auf `/#blog`.
4. **Suchmaschinen-Angaben** je Beitragsseite: eigener Titel und eigene Beschreibung.

### Betroffene Stellen (technisch)

- `src/landing-themes/theme-mirror-site/template.html` — Kachel-Links
- `src/landing-themes/theme-mirror-site/meta.json` — neue Slots `blog_post_N_body`
- `landing-server/server.ts` (+ kompiliertes `server.js`) — neue Route `/blog/:n`, Renderer analog `renderLegal`
- Danach: Landing-Server-Sync/Deploy, damit die Änderung live ist

## Noch offen

Der zweite Screenshot (fehlerhafter Fußbereich / Impressum) fehlt noch. Sobald er da ist, ergänze ich den Fix im selben Zug.
