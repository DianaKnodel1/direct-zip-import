# CSV-Export für Mitarbeiter und Bewerber

Ergänzt beide bestehenden Listen um einen Button „CSV exportieren“. Listen, Filter, Suche, Statuslogik, Datenbank und Berechtigungen bleiben unverändert.

## Mitarbeiter

- Button „CSV exportieren“ neben Suche/Tabs in der Mitarbeiterliste.
- Exportiert exakt die aktuell sichtbare Menge (aktiver Tab + Suchtext), also die bereits berechnete gefilterte Liste — keine neue Abfrage, keine zweite Filterlogik.

## Bewerber

- Button „CSV exportieren“ mit kleinem Dropdown-Menü (kein Dialog).
- Menüeinträge = die bereits vorhandenen Statusgruppen: Alle, Eingegangen, Termin gebucht, Interview, Nicht erschienen, Abgesagt, Zusage erteilt, Abgelehnt, Onboarded.
- Die Auswahl gilt nur für den Export; der aktive Tab der Liste ändert sich nicht.
- Zusätzlich wirken die bestehenden Einschränkungen weiter: Mandanten-/Firmenauswahl, Archiv-Schalter und Suchtext. Die gewählte Statusgruppe wird auf dieselbe bereits gefilterte Grundmenge angewendet, die auch die Liste nutzt.

## CSV-Inhalt

Ausschließlich diese Spalten, in dieser Reihenfolge:
Vorname, Nachname, E-Mail, Telefon, Straße, PLZ, Ort, Land.

Keine Status-, Firmen-, KYC-, Vertrags- oder Auftragsdaten.

Quelle der Felder (bereits geladen, keine neuen Abfragen):

- Bewerber: `first_name`/`last_name` (bzw. Aufteilung aus `full_name`), `email`, `phone`, `address`, `postal_code`, `city`.
- Mitarbeiter: Profil `full_name` (aufgeteilt), Konto-E-Mail, `phone`, `street` (Rückfall `address`), `zip_code`, `city`; fehlt etwas im Profil, greift die verknüpfte Bewerbung als Rückfall.

Hinweis: Ein echtes Länderfeld existiert in den Daten nicht. Die Spalte „Land“ wird deshalb mit ausgegeben, bleibt aber leer, solange kein Wert vorhanden ist. Alternativ kann sie fix mit „Deutschland“ vorbelegt werden — sag Bescheid, wenn das gewünscht ist.

## Format

- UTF-8 mit BOM, Semikolon als Trennzeichen (Excel-Standard in DE).
- Werte mit Semikolon, Komma, Anführungszeichen oder Zeilenumbruch werden korrekt gequotet und escaped; leere Werte bleiben leer.
- Dateiname mit aktuellem Datum: `mitarbeiter-2026-08-28.csv`, `bewerber-alle-2026-08-28.csv`, `bewerber-zusage-erteilt-2026-08-28.csv`.
- Bei 0 Treffern: Toast „Keine Datensätze zum Exportieren vorhanden.“, kein Download.

## Sicherheit

Der Export nutzt ausschließlich die Daten, die der angemeldete Admin ohnehin schon geladen und in der Liste sieht (RLS/Mandantentrennung greifen unverändert). Kein Server-Zugriff mit erhöhten Rechten, keine neuen Endpunkte, keine Änderung an Rechten.

## Technische Umsetzung

- Neu: `src/lib/csv-export.ts` — reine Hilfsfunktionen: `toCsv(rows)` (BOM, Semikolon, Escaping) und `downloadCsv(filename, rows)` (Blob + temporärer Link).
- `src/routes/admin.mitarbeiter.tsx`: Row-Mapping um die Adressfelder aus `profiles`/`applications` erweitern (nur zusätzliche Felder, bestehende bleiben unberührt); Export-Button rendert und exportiert `filtered`.
- `src/routes/admin.bewerbungen.tsx`: Row-Mapping um Adressfelder aus `applications` erweitern; `DropdownMenu` mit den `GROUPS`-Einträgen; Export wendet `groupOf(phase) === key` (bzw. alle) plus vorhandenen Suchtext auf `scoped` an — dieselbe Funktion wie die Liste, nur mit dem gewählten Statusschlüssel statt `tab`.
- Danach Build und Typecheck.

## Verifizierung

Manuelle Prüfung über den Browser: Mitarbeiter (alle / mit Filter), Bewerber (Alle, Zusage erteilt, Abgelehnt, Onboarded, Status + Firma, Status + Suche), Umlaute, Sonderzeichen/Kommas in Adressen, leere Felder, 0-Treffer-Meldung.
