# Landing-Page-Löschen entschärfen + Mandant im CSV-Export

## Teil 1 — Warum das Löschen fehlschlägt

**Screenshot 1 (Landing Page löschen):**
Die Fehlermeldung kommt direkt aus der Datenbank:
`update or delete on table "availability_schedules" violates foreign key constraint "interview_appointments_schedule_id_fkey"`.

Kette:
```text
landing_pages  --(löschen kaskadiert)-->  availability_schedules (Terminkalender der Landing)
availability_schedules  <--(gesperrt: ON DELETE RESTRICT)--  interview_appointments (gebuchte Termine)
```
Die Landing-Page hat einen Buchungskalender, und zu diesem Kalender existieren bereits gebuchte Interview-Termine. Die Termine sind bewusst gegen Löschen geschützt (Historie), deshalb bricht die ganze Löschung ab. Es ist also kein Bug im Code, sondern ein Datenschutz-Mechanismus — aber aktuell ohne verständliche Meldung und ohne Ausweg in der UI.

**Screenshot 2 (Mandant löschen):**
Das ist bereits die gewünschte, freundliche Variante: die App prüft vorher die Verknüpfungen (2 Mitarbeiter, 2 unterschriebene Verträge, 3 Vertragsvorlagen) und erklärt, was zu tun ist. Hier ist nichts kaputt — die Empfehlung „Deaktivieren statt Löschen" ist korrekt.

## Möglichkeiten für die Landing-Page

1. **Deaktivieren statt löschen** — Landing offline nehmen, alles bleibt erhalten. Sofort möglich, kein Datenverlust, Domain-Eintrag bleibt aber bestehen.
2. **Löschen und Termine behalten (Empfehlung)** — vor dem Löschen wird der Kalender von der Landing gelöst (Landing-Bezug entfernt, Kalender auf inaktiv). Die gebuchten Termine bleiben vollständig erhalten und weiterhin in „Mitarbeiter-Termine" sichtbar; die Landing verschwindet sauber.
3. **Alles löschen inkl. Termine** — kompletter Verlust der Terminhistorie dieser Landing. Nicht empfohlen.

**Beste Lösung:** Variante 2 als Standard, mit einem klaren Hinweis-Dialog vorher (nach Vorbild der Mandanten-Meldung), plus „Deaktivieren" als angebotene Alternative im selben Dialog.

## Umsetzung Teil 1

- Neue Vorabprüfung beim Löschen: zählt Kalender + gebuchte Termine der Landing und liefert diese Zahlen zurück.
- Löschdialog zeigt bei Treffern: „Diese Landing hat X gebuchte Termine. Die Termine bleiben erhalten, die Landing wird entfernt." mit den Buttons „Trotzdem löschen (Termine behalten)" und „Nur deaktivieren".
- Beim Bestätigen: Kalender von der Landing lösen und inaktiv setzen, danach Landing löschen — in einem Schritt serverseitig, weiterhin über die Admin-Rechte des angemeldeten Nutzers (RLS unverändert, keine Service-Role).
- Datenbankfehler, die trotzdem auftreten, werden in Klartext übersetzt statt als roher Constraint-Text angezeigt.

## Teil 2 — Mandant im CSV-Export

Zusätzliche Spalte **„Mandant"** in beiden Exporten, eingefügt als letzte Spalte:
`Vorname; Nachname; E-Mail; Telefon; Straße; PLZ; Ort; Land; Mandant`

- **Bewerber:** Mandantenname aus der bereits geladenen `tenant_id` der Bewerbung (Namensliste ist auf der Seite schon vorhanden).
- **Mitarbeiter:** Mandantenname aus `tenant_id` des Profils, Rückfall auf die verknüpfte Bewerbung. Dafür wird die Mandanten-Namensliste auf der Mitarbeiterseite ebenfalls geladen (gleiche Abfrage wie bei den Bewerbern).
- Ohne Zuordnung bleibt das Feld leer.
- Format (UTF-8 mit BOM, Semikolon, Escaping) und Dateinamen bleiben unverändert.

## Technische Details

- `src/lib/csv-export.ts`: `CONTACT_HEADERS` um „Mandant" erweitern, `ContactRow` um `tenant`.
- `src/routes/admin.bewerbungen.tsx`: im Export-Mapping `tenant` aus der vorhandenen Tenant-Map setzen.
- `src/routes/admin.mitarbeiter.tsx`: Tenants laden (`tenants: id, name`), Row-Mapping um `tenantId` (Profil → Bewerbung) erweitern, Export-Mapping um `tenant`.
- `src/lib/landing-pages.functions.ts`: neue Prüf-Funktion (Kalender/Termine zählen) und `deleteLandingPage` um „Kalender lösen, dann löschen" erweitern.
- Landing-Generator-UI (`admin.tsx`-Bereich mit der Landings-Tabelle): Bestätigungsdialog mit den beiden Optionen.
- Abschluss: Build und Typecheck.
