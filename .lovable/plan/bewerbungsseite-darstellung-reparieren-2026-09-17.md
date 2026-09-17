# Bewerbungsseite: Darstellung reparieren

## Stand nach dem Test

Ich habe auf cr-beratung.solutions/bewerben eine echte Testbewerbung abgeschickt:

- Cookie-Hinweis bestätigt → das Meta-Pixel (ID 2113512246264776) wird geladen.
- Formular abgeschickt → Weiterleitung auf `/danke` mit der Markierung `lead=1`,
  d. h. das Lead-Ereignis wurde bereits beim Absenden gemeldet und wird auf der
  Danke-Seite nicht doppelt gezählt. Der Calendly-Termin-Link wird korrekt
  mitgegeben.

Der von deinem Werbe-Fachmann gemeldete Fehler ist damit behoben.

## Was noch kaputt ist

Das Bewerbungsformular erscheint zwar, aber komplett ohne Gestaltung: nackte
Eingabefelder, ein kleines „×" oben links, und die Seite zeigt zwei Fußzeilen
untereinander. Ursache: Die Seite liefert den Formularbereich als Einblend-Fenster
aus, dessen Gestaltungsregeln in der ausgelieferten Fassung fehlen, und die
Grundfassung der Seite bringt zusätzlich eine eigene Fußzeile mit.

## Umsetzung

1. Formularbereich auf `/bewerben` als normalen Seitenabschnitt ausliefern statt
   als Einblend-Fenster: kein „×"-Schließen, keine Überlagerung, volle Breite im
   Seitenraster.
2. Die vollständigen Gestaltungsregeln des Formulars (Feld-Abstände, Rahmen,
   Schaltflächen, Schriftgrößen, Mobilansicht) sicher mit ausliefern, damit die
   Felder wie auf der Startseite aussehen.
3. Doppelte Fußzeile unterdrücken: auf `/bewerben` nur die vollständige Fußzeile
   des Themes rendern.
4. Gleiche Prüfung für `/danke` durchführen.
5. Nach dem Ausrollen erneut automatisiert testen: Seite ansehen, Formular
   ausfüllen, absenden, Weiterleitung mit `lead=1` und einwandfreies Aussehen
   bestätigen.

## Technische Details

- Betroffen: `landing-server/server.ts` und die parallel gepflegte
  `landing-server/server.js` (Modal-Wrapper und eingespritzte Styles im
  `/bewerben`-Rendering), dazu ggf. `src/landing-themes/_shared/form-section.*`.
- Fußzeilen-Dopplung über die bestehende Prüfung `hasRichThemeFooter()` lösen.
- Die Lead-Logik (`leadFired`, `lead=1`, `buildPixelBlock`) bleibt unverändert.
- Ausrollen wie gehabt: `cd /opt/apps/portal && git pull && bash scripts/deploy.sh`.
