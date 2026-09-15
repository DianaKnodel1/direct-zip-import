# Eigene Adressen für Bewerbung und Danke-Seite

Ziel: Für Meta-/Facebook-Anzeigen bekommt jede Landing Page zwei echte Unterseiten
statt Sprungmarken und Pop-ups:

- `kunde.de/bewerben` — das Bewerbungsformular als eigene Seite
- `kunde.de/danke` — die Danke-Seite nach erfolgreichem Absenden

Der Ablauf danach bleibt inhaltlich gleich: Bei Fast-Track erscheint auf der
Danke-Seite der Terminbereich, über den die Bewerberin bzw. der Bewerber den
Termin bucht (Calendly-Buchung wie bisher).

## Ablauf nach der Umstellung

```text
Anzeige  →  kunde.de/bewerben   (Formular, eigene Adresse)
                 │ absenden
                 ▼
            kunde.de/danke      (Danke-Text + Terminauswahl)
                 │ "Termin buchen"
                 ▼
            Terminbuchung (Calendly) wie bisher
```

## Was sich ändert

1. **Startseite:** Alle "Jetzt bewerben"-Schaltflächen führen auf `/bewerben`
   statt nach unten zum Formular. Der Formularbereich auf der Startseite entfällt
   nicht sofort — er bleibt als Rückfallebene erhalten, wird aber nicht mehr
   verlinkt (alte Anzeigen mit `#bewerbung` laufen weiter).
2. **Neue Seite `/bewerben`:** Kopf- und Fußbereich im jeweiligen Theme-Design,
   darin das bekannte Formular.
3. **Neue Seite `/danke`:** Bestätigungstext, und — sofern eine Terminbuchung
   nötig ist — direkt darunter die Terminauswahl mit der Schaltfläche
   "Termin buchen". Ohne Terminbuchung nur Bestätigung plus Link zurück.
4. **Nach dem Absenden** wird nicht mehr das Pop-up geöffnet, sondern auf
   `/danke` weitergeleitet. Die für die Terminbuchung nötige Kennung wird dabei
   mitgegeben, damit die Terminauswahl dort weiß, um welche Bewerbung es geht.
5. **Alte Adressen** `/#bewerbung` leiten auf `/bewerben` weiter.

## Meta-Pixel

- Im Landing-Generator kommt ein neues Feld **Facebook-Pixel-ID** (pro Landing Page).
- Ist es gefüllt, lädt das Pixel auf allen Seiten der Landing Page und meldet auf
  `/danke` das Ereignis **Lead**.
- Ist es leer, ändert sich nichts — kein Pixel, kein Banner.

**Einwilligung (wichtig, muss vor dem Scharfschalten entschieden werden):**
Die Landing Pages richten sich an Besucher in Deutschland/EU. Empfehlung: ein
schlanker Cookie-Hinweis auf der Landing Page, der nur in einwilligungspflichtigen
Regionen erscheint; das Pixel lädt erst nach Zustimmung, Ablehnen ist genauso
einfach wie Zustimmen und später änderbar. Anzeigenklicks werden unabhängig davon
gezählt, nur die Messung der zustimmenden Besucher kommt hinzu.
Alternative ohne Banner: Das Pixel bleibt in diesen Regionen komplett aus —
dann ist die Messung dort nicht möglich. Bitte eine der beiden Varianten wählen;
ich baue das Feld in beiden Fällen, aktiviere aber nur die gewählte Variante.
Die Datenschutzerklärung der Landings wird um Meta-Pixel, Zweck und Widerruf ergänzt.

## Technische Umsetzung

- `landing-server/server.ts` (+ kompiliertes `server.js`): neue Routen `/bewerben`
  und `/danke`; beide nutzen das Theme-Layout (Kopf/Fuß, `style.css`, `script.js`).
  Rückleitung `/#bewerbung` → `/bewerben`.
- `src/landing-themes/_shared/form-section.js`: Nach erfolgreichem POST kein
  `showModal()` mehr, sondern `location.assign('/danke?...')` mit der
  Buchungs-Kennung aus `redirect_url`. `renderBookingInline()` wird auf der
  Danke-Seite aus denselben Daten aufgerufen; die Pop-up-Variante bleibt als
  Rückfall für Fälle ohne Kennung.
- CTA-Ziele: `href="#bewerbung"` in den Theme-Templates und in
  `FORM_OVERRIDES`/`landing-themes.ts` auf `/bewerben` umstellen.
- `src/lib/landing-generator.functions.ts` + Landing-Generator-Oberfläche:
  Feld `meta_pixel_id` im Branding, Pixel-Snippet und Consent-Gate im gerenderten
  Kopfbereich, `Lead`-Ereignis nur auf `/danke`.
- Datenbank: keine Schemaänderung nötig — die Pixel-ID liegt im vorhandenen
  `branding`-Feld der Tabelle `landing_pages`.
- Danach: `bash scripts/deploy.sh` und Landing-Sync, damit die Seiten live gehen.

## Prüfungen vor dem Abschluss

- `/bewerben` und `/danke` liefern auf einer Testdomain HTTP 200 im Theme-Design.
- Testbewerbung: Weiterleitung auf `/danke`, Terminauswahl erscheint, Buchung geht durch.
- Nur ein Fußbereich, Impressum- und Datenschutzlinks vorhanden.
- Ohne Pixel-ID wird weder Pixel noch Hinweis geladen.
