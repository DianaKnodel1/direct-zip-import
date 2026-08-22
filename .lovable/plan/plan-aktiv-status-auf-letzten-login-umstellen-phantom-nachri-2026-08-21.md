# Plan: Aktiv-Status auf letzten Login umstellen & Phantom-Nachrichten als Systemnachrichten

## Ziel 1: „Aktiv"-Status soll den letzten Login zeigen

**Befund (`src/routes/admin.chat.tsx`, Zeile 994):** Die Statuszeile nutzt `conv.lastSeenAt ?? conv.lastSignInAt`. `last_seen_at` ist ein Heartbeat, der alle 60 s aktualisiert wird, solange der Mitarbeiter die App offen hat (`src/hooks/use-presence.ts` → `updateLastSeen`). Der Status zeigt daher die letzte App-Nutzung, nicht den letzten Login.

**Änderungen:**
- Statuszeile in der Admin-Chatliste auf `conv.lastSignInAt` (letzter Login aus `auth.users` via RPC `get_last_sign_ins`) umstellen. Der grüne „● Online"-Punkt (Live-Presence) bleibt unverändert.
- Fehlerlogik anpassen: Der Hinweis „Login-/Aktivitätsstatus nicht verfügbar" soll schon erscheinen, wenn die Login-Quelle (`signInError`) ausfällt – aktuell nur, wenn beide Quellen fehlschlagen.
- Fallback-Texte bleiben: „Noch nie eingeloggt" / „Status unbekannt".
- Prüfen, ob die RPC-Funktion `get_last_sign_ins` auf der angebundenen Datenbank existiert (sie liegt in `supabase/manual-migrations/20260903000000_last_sign_in_and_last_seen.sql`); falls nicht, einspielen.

## Ziel 2: Automatische Nachrichten nicht mehr als persönliche Nachrichten des Teamleiters zeigen

**Befund:** Fünf DB-Trigger erzeugen Chat-Nachrichten mit dem Teamleiter als `sender_id`, ohne `is_system = true` zu setzen. Der Mitarbeiter-Chat (`src/routes/_employee/chat.tsx`, `isSystemMessage`) rät Systemnachrichten nur anhand weniger Emoji-Präfixe – diese Texte fallen durch und erscheinen wie persönliche Nachrichten vom Admin:

- `send_welcome_chat_message` → „Hallo <Name>! Willkommen im Team!"
- `send_system_chat_on_profile_change` → „Vertrag unterschrieben!", „Einführung abgeschlossen!"
- `send_chat_on_kyc_change` → „Verifizierung bestätigt!"
- `send_chat_on_task_assignment` → „Neuer Auftrag: <Titel>"
- SMS-Weiterleitung (`trg_forward_inbound_sms_to_chat` u. a.) → „📩 SMS Code: …"

**Änderungen:**
- Neue SQL-Migration (Konvention des Projekts: `supabase/manual-migrations/`), die:
  1. `chat_messages.is_system` absichert (`ADD COLUMN IF NOT EXISTS`),
  2. alle fünf Trigger-Funktionen per `CREATE OR REPLACE` so aktualisiert, dass sie `is_system = true` setzen,
  3. Bestandsdaten kennzeichnet: `UPDATE chat_messages SET is_system = true` für die bekannten Auto-Texte („Vertrag unterschrieben!", „Einführung abgeschlossen!", „Verifizierung bestätigt!", `LIKE 'Neuer Auftrag: %'`, `LIKE '📩 SMS Code: %'`, Willkommensmuster) – eingegrenzt auf Zeilen, deren Absender Teamleiter des Empfängers oder Admin ist.
- Migration auf die angebundene Datenbank anwenden (Zugangsdaten aus der projekteigenen `.env`).
- Frontend-Absicherung in `src/routes/_employee/chat.tsx`: `isSystemMessage`-Fallback erweitern, damit die bekannten Auto-Texte auch dann als Systemnachricht (zentriert, neutral, ohne Avatar/Name des Teamleiters) gerendert werden, wenn das `is_system`-Flag fehlt.
- `src/components/FloatingChat.tsx`: gleiche Systemnachrichten-Behandlung ergänzen (dort werden die Texte derzeit ebenfalls als normale Teamleiter-Nachrichten angezeigt).

## Nicht geändert

- Platzhalter-Box „👋 Hallo! Ich bin für dich da …" im leeren Mitarbeiter-Chat bleibt wie sie ist (Entscheidung des Users).
- Live-„Online"-Punkt und Presence-Heartbeat bleiben bestehen.

## Verifikation

- Build-Log prüfen (build OK).
- Admin-Chat: Statuszeile zeigt „Aktiv vor …" basierend auf dem letzten Login; Online-Punkt unverändert.
- Mitarbeiter-Chat & FloatingChat: Auto-Nachrichten erscheinen als neutrale Systemnachrichten, nicht mehr als persönliche Nachrichten des Admins.
