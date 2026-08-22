# Online-Zeit, Chat-Mehrfachauswahl, Domain-Monitoring

## 1. Online-Zeit im Admin-Chat korrigieren

**Befund:** Der Heartbeat (`profiles.last_seen_at`, alle 60 s bei geöffneter App) und die Spalte existieren inzwischen auf der Datenbank — aber die Anzeige nutzt nur den letzten Login. Da Mitarbeiter wochenlang eingeloggt bleiben, wirkt der Status veraltet.

**Fix (`src/routes/admin.chat.tsx`):**
- Statuszeile zeigt „Zuletzt aktiv …" auf Basis von `last_seen_at` (Heartbeat), Fallback: letzter Login.
- Letzter Login erscheint zusätzlich als Tooltip/Zweitinfo, geht nicht verloren.
- „Status unbekannt"-Warnung nur noch, wenn beide Quellen ausfallen.

## 2. Mehrere Chats auswählen & deaktivieren

**Befund:** Ausblenden (`admin_hidden_at`) existiert pro Chat bereits — es fehlt nur die Mehrfachauswahl.

**Umsetzung (`src/routes/admin.chat.tsx`):**
- „Auswählen"-Modus in der Chat-Liste: Checkboxen pro Chat, „Alle auswählen".
- Aktionsleiste bei Auswahl: **Ausblenden** (Tab „Aktiv") bzw. **Einblenden** (Tab „Ausgeblendet") für alle markierten Chats — nutzt die vorhandene `admin_hidden_at`-Logik, keine DB-Änderung nötig.

## 3. Domain-Monitoring überarbeiten

**Befund:** Der Cron (`domain-health-cron`, alle 5 Min) pingt nur Domains aus `tenants` und schreibt bei Ausfall nur ins Activity-Log — das sieht man nicht, ohne die Log-Seite zu öffnen. Domains aus `landing_pages` (die eigentlichen Landing-Domains!) fehlen in der Übersicht komplett.

**Umsetzung:**

1. **Neue Tabelle `domain_watchlist`** (Migration): manuell hinzugefügte Domains (domain, Notiz, created_at). Admin-only per RLS, mit GRANTs.
2. **Neue Tabelle `domain_check_results`** (Migration): letzter Prüfstand pro Domain (domain, Quelle, Status, HTTP-Code, Latenz, Fehler, geprüft_am). Der Cron schreibt hier rein — der Admin sieht den Status sofort, ohne dass die Seite bei jedem Öffnen 20+ Live-Pings macht.
3. **Cron erweitern** (`src/routes/api/public/domain-health-cron.ts`): prüft künftig `tenants`-Domains + `landing_pages`-Domains + Watchlist; schreibt Ergebnisse in `domain_check_results`; Activity-Log nur noch bei Status-Wechsel auf „down" (kein Log-Spam alle 5 Min).
4. **Warn-Banner im Admin-Dashboard** (`admin.index.tsx`): roter Hinweis „⚠ 2 Domains nicht erreichbar" mit Link zur Übersicht, sobald ein Eintrag in `domain_check_results` down/no_landing ist. Das löst dein Kernproblem: Du siehst Ausfälle sofort beim Öffnen des Panels.
5. **Seite `/admin/domains` ausbauen**: zeigt alle Domains aus allen drei Quellen (gekennzeichnet: Tenant / Landing / Manuell), Eingabefeld „Domain hinzufügen" mit Sofort-Check, Entfernen-Button für Watchlist-Einträge, „Alle jetzt prüfen"-Button.

**Bewusst nicht dabei:** E-Mail-/SMS-Alarm bei Ausfall (sag Bescheid, wenn du das willst — dafür nutzen wir die vorhandene Mail-Infrastruktur).

## Migrationen

Eine neue Datei in `supabase/manual-migrations/` mit beiden Tabellen (inkl. GRANT + RLS). Du spielst sie wie gehabt im SQL-Editor ein; bis dahin fallen Banner und Cron-Erweiterung graceful auf Live-Pings zurück.

## Reihenfolge

1 → 2 (sofort wirksam, keine Migration), dann 3 (Migration + Code).
