# Auftrags-Lücken schließen, Statistik fixen, Ads-Readiness-Check

Drei Baustellen aus deinen Fragen. Entscheidungen: Ablehnung bleibt erneut einreichbar, alle drei Lücken werden geschlossen, voller Ads-Check.

## 1. Auftrags-Ablehnung & Genehmigung — Lücken schließen

**Datei:** `src/routes/admin.assignments.$assignmentId.tsx` (Funktion `updateStatus`)

- **Phantom-Vergütung fixen:** Beim Ablehnen werden alle `user_transactions` mit Status `ausstehend` dieses Auftrags gelöscht. Der Mitarbeiter sieht dann kein ausstehendes Geld mehr für abgelehnte Arbeit.
- **Doppelzählung fixen:** Beim Genehmigen wird eine vorhandene `ausstehend`-Transaktion auf `genehmigt` umgestellt (Update), statt eine zweite Transaktion anzulegen. Nur wenn keine existiert, wird eine neue angelegt. Der „Ausstehend"-Betrag auf der Verdienst-Seite stimmt dann.
- **Benachrichtigung:** Bei Ablehnung, Nachbesserung und Genehmigung wird ein Eintrag in `notifications` geschrieben (user_id, type `assignment_status`, Titel + Nachricht inkl. Admin-Kommentar). Der Mitarbeiter sieht es sofort an der Glocke.

**Erneut eingereichte Aufträge sichtbar machen** — `src/routes/admin.revisions.tsx`:
- Neue Sektion „Erneut eingereicht": Aufträge mit Status `eingereicht`, die bereits `step_feedback`-Einträge haben (d.h. sie wurden früher abgelehnt oder zur Nachbesserung markiert). So geht eine erneute Einreichung nach Ablehnung nicht mehr im allgemeinen Stapel unter. Kein Datenbank-Change nötig.

## 2. Statistiken: fehlende „classic"-Bewerbungen

**Datei:** `src/lib/landing-cohorts.functions.ts`

- Der Filter `flow_type IN (broker, fast)` schließt alle `classic`-Bewerbungen aus — das ist der Standardwert, wenn eine Landingpage keinen Flow-Typ gesetzt hat. Der Filter wird entfernt (alle Bewerbungen mit `is_test=false` zählen).
- Kurzer Hinweistext unter dem Funnel in `admin.statistiken.tsx`: „Kohorte = Bewerbungstag. Termine/Interviews späterer Tage reifen in der Tageszeile nach." — damit die Logik „Buchung heute für den 27.08. zählt heute" nachvollziehbar bleibt.

Die Kohorten-Logik selbst bleibt unverändert — sie ist korrekt: „Termin wahrgenommen" wird erst gesetzt, wenn das Interview tatsächlich abgeschlossen wurde.

## 3. Ads-Readiness-Check (vor neuer Traffic-Schaltung)

- **Offene Migrationen auflisten** (müssen im Supabase SQL-Editor eingespielt werden, ich habe keinen DB-Zugang): `20260901000000` (Chat is_system), `20260904000000` (Auto-Nachrichten), `20260905000000` (Domain-Monitoring) + der pg_cron-Job `20260603000000_domain_health_cron.sql` mit echtem CRON_SECRET. Ohne diese läuft das Domain-Alarmbanner nicht.
- **End-to-End-Testlauf** des Bewerberpfads per Browser-Automatisierung gegen die Preview: Testbewerbung (is_test) → Terminbuchung → KI-Interview → Einladungs-Mail-Status → Registrierung. Pro Schritt Ampel-Status.
- **Abschluss-Report:** Klare Go/No-Go-Einschätzung pro Funnel-Stufe mit konkreten Befunden, bevor Budget ausgegeben wird.

## Technische Details

- `updateStatus` in `admin.assignments.$assignmentId.tsx` bekommt drei zusätzliche Blöcke: Transaktions-Bereinigung bei `abgelehnt`, Transaktions-Umstellung bei `genehmigt`, notifications-Insert bei `abgelehnt`/`nachbesserung`/`genehmigt`.
- notifications-Spalten (verifiziert): `user_id`, `type`, `title`, `message`, `read`, `created_at`.
- Revisions-Erkennung ohne Migration: Join auf `step_feedback` (Einträge bleiben nach erneuter Einreichung als `resolved=true` erhalten).
- Statistik: nur der `.in("flow_type", ...)`-Filter entfällt; Rest der Funktion unverändert.
- Kein Schema-Change nötig; alle DB-Änderungen laufen über die bestehenden Tabellen.
