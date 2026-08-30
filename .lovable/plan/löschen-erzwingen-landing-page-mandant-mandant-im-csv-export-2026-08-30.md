# Löschen erzwingen (Landing Page & Mandant) + Mandant im CSV-Export

## Teil 1 — Warum das Löschen aktuell blockiert

**Landing Page (Screenshot 1):** Die Datenbank meldet
`... "availability_schedules" violates foreign key constraint "interview_appointments_schedule_id_fkey"`.

```text
landing_pages  --(löschen kaskadiert)-->  availability_schedules (Buchungskalender)
availability_schedules  <--(gesperrt)--   interview_appointments (gebuchte Termine)
```
Die Landing hat einen Buchungskalender mit bereits gebuchten Terminen. Termine sind gegen Löschen geschützt, deshalb bricht alles ab.

**Mandant (Screenshot 2):** Kein Datenbankfehler, sondern eine eingebaute Vorabprüfung: solange Bewerbungen, Mitarbeiter, Verträge, Vertragsvorlagen oder Dokumente am Mandanten hängen, wird gar nicht erst gelöscht.

## Was passiert, wenn trotzdem gelöscht wird?

**Landing Page löschen (erzwungen):**
- Die Landing verschwindet aus dem Generator und ist unter ihrer Domain nicht mehr erreichbar (der Landing-Server findet keinen Eintrag → 404).
- Der Buchungskalender der Landing wird von ihr gelöst und deaktiviert — **gebuchte Termine bleiben vollständig erhalten** und weiterhin unter „Mitarbeiter-Termine" sichtbar.
- Bewerbungen, die über diese Landing kamen, bleiben erhalten; nur der Herkunfts-Verweis („kam von Landing X") ist danach leer.
- Verknüpfte Fast-Track-Landings verlieren die Verknüpfung, laufen aber weiter.
- Nicht wiederherstellbar: Theme-Konfiguration, Slots, Branding dieser Landing. Der Cloudflare-DNS-Eintrag bleibt bestehen und muss ggf. separat entfernt werden.

**Mandant löschen (erzwungen):**
- Alle Daten des Mandanten werden vorher vom Mandanten **gelöst**, nicht gelöscht: Bewerbungen, Mitarbeiterprofile, Verträge, Vertragsvorlagen und Dokumente bleiben bestehen, stehen danach aber ohne Mandantenzuordnung da („kein Mandant").
- Folge im Alltag: diese Datensätze tauchen in Mandantenfiltern nicht mehr auf, Mandanten-Auswertungen und der neue CSV-Mandantenwert sind für sie leer, E-Mail-Absender/Branding fallen auf die Standardwerte zurück.
- Endgültig gelöscht werden die mandantengebundenen Einstellungen selbst: SMTP-/Absenderkonfiguration, Chat-FAQ, Bot-Einstellungen, Calendly-Konten, Bounce-/Sperrlisten und Mail-Warteschlangen dieses Mandanten.
- Nicht rückgängig machbar. Wo nur „aufräumen" gewollt ist, bleibt **Deaktivieren** die bessere Wahl.

## Umsetzung Teil 1

**Landing Page (Landing-Generator):**
- Vorabprüfung zählt Kalender und gebuchte Termine der Landing.
- Löschdialog statt sofortigem Löschen: zeigt die Zahlen und die Folgen in Klartext, mit drei Optionen: „Löschen (Termine behalten)", „Nur deaktivieren", „Abbrechen".
- Beim Löschen serverseitig in einem Schritt: Kalender von der Landing lösen und inaktiv setzen, Fast-Track-Verknüpfungen lösen, danach Landing löschen.

**Mandant (Mandantenverwaltung):**
- Bestehende Prüfung bleibt, wird aber nicht mehr zur Sackgasse: die Meldung wird ein Dialog mit „Deaktivieren" (Standardempfehlung) und „Trotzdem löschen".
- „Trotzdem löschen" verlangt zur Sicherheit die Eingabe des Mandantennamens und löst dann zuerst alle Verknüpfungen (tenant_id leeren) auf Bewerbungen, Profilen, Verträgen, Vertragsvorlagen, Dokumenten und Landings, bevor der Mandant gelöscht wird.
- Bleibt trotzdem ein Datenbankfehler übrig, wird er in Klartext übersetzt statt als roher Constraint-Text angezeigt.

Beides läuft weiterhin mit den Rechten des angemeldeten Admins (RLS/Mandantentrennung unverändert, keine Service-Role).

## Teil 2 — Mandant im CSV-Export

Zusätzliche letzte Spalte **„Mandant"**:
`Vorname; Nachname; E-Mail; Telefon; Straße; PLZ; Ort; Land; Mandant`

- **Bewerber:** Mandantenname über die bereits geladene `tenant_id` der Bewerbung (Namensliste ist auf der Seite vorhanden).
- **Mitarbeiter:** Mandantenname über `tenant_id` des Profils, Rückfall auf die verknüpfte Bewerbung; die Mandanten-Namensliste wird auf der Mitarbeiterseite mit derselben Abfrage geladen.
- Ohne Zuordnung bleibt das Feld leer.
- Format (UTF-8 mit BOM, Semikolon, Escaping) und Dateinamen bleiben unverändert.

## Technische Details

- `src/lib/csv-export.ts`: `CONTACT_HEADERS` um „Mandant", `ContactRow` um `tenant`.
- `src/routes/admin.bewerbungen.tsx`: Export-Mapping um `tenant` aus der vorhandenen Tenant-Map.
- `src/routes/admin.mitarbeiter.tsx`: `tenants (id, name)` laden, Row-Mapping um `tenantId` (Profil → Bewerbung), Export-Mapping um `tenant`.
- `src/lib/landing-pages.functions.ts`: neue Prüf-Funktion (Kalender/Termine zählen); `deleteLandingPage` löst Kalender und Verknüpfungen vor dem Löschen.
- Landing-Generator-UI: Bestätigungsdialog mit den drei Optionen.
- `src/routes/admin.tenants.tsx`: `deleteTenant` wird Dialog mit Deaktivieren/Force-Delete inkl. Namenseingabe und Entkopplungsschritt.
- Abschluss: Build und Typecheck.
