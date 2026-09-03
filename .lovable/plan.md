# Analyse: „Zuletzt aktiv" im Chat

Nur Analyse — noch keine Codeänderung durchgeführt.

## 1. Chat-Architektur

- **Oberflächen:** `src/routes/admin.chat.tsx` (Admin-Postfach), `src/routes/_employee/chat.tsx` (Mitarbeiter-Vollansicht), `src/components/FloatingChat.tsx` (Widget).
- **Tabellen:** `chat_messages` (Nachrichten), `chat_conversations` (Status, Eskalation, Admin-Unread, Notiz), `profiles` (Name, Mandant, `last_seen_at`, `leader_online`), `auth.users` (`last_sign_in_at`), `user_roles` (Rollen).
- **Laden:** Admin holt die Liste über RPC `list_chat_conversations`, mit Fallback auf paginierte Rohabfrage (`admin.chat.tsx:172-197`); Verlauf pro Kontakt separat (`admin.chat.tsx:333-345`).
- **Realtime:** Channel `admin-chat-unified` auf `chat_messages` (INSERT) und `chat_conversations` (UPDATE), plus Resync bei Tab-Wechsel/Fokus/Online und 25-Sekunden-Fallback-Poll (`admin.chat.tsx:717-828`, Helfer in `src/lib/chat-sync.ts`).
- **Online-Erkennung:** getrennter Realtime-Presence-Channel (`src/hooks/use-presence.ts`), gelesen über `useOnlineUsers()` → grüner Punkt „● Online".

## 2. Woher „Zuletzt aktiv" kommt

- Text erzeugt in `admin.chat.tsx:308-318` (`formatLastActive`), gerendert in `admin.chat.tsx:1169-1173`.
- Zeitwert = jüngster von **drei** Quellen (`admin.chat.tsx:321-323`, `latestTimestamp` aus `chat-sync.ts:104-113`):
  1. `profiles.last_seen_at` (Heartbeat),
  2. `auth.users.last_sign_in_at` (RPC `get_last_sign_ins`),
  3. `lastFromEmployeeAt` = **Zeit der letzten Nachricht** des Mitarbeiters.
- Geladen einmalig per Server-Function `getLastSignIns` (`src/lib/last-sign-ins.functions.ts`) am Ende von `loadConversations()` (`admin.chat.tsx:288-303`).
- Heartbeat: `usePresenceBroadcast()` global in `src/routes/__root.tsx:151`, schreibt alle 60 s über `updateLastSeen` (`src/lib/presence.functions.ts:17-34`) `profiles.last_seen_at`. Er läuft für jeden eingeloggten User, unabhängig von Chat oder Navigation; beim Sichtbarwerden des Tabs zusätzlich ein Extra-Beat. Beim Senden/Empfangen einer Nachricht wird der Wert **nicht** gesetzt.
- Zweite, völlig andere Anzeige auf Mitarbeiterseite: `src/hooks/use-team-leader.ts:80,105` nutzt nur das manuell umschaltbare Flag `profiles.leader_online` (Umschalter in `admin.chat.tsx:983-996`) — kein Bezug zu Heartbeat oder Presence.

## 3. Backend / Datenbank

- Spalte `profiles.last_seen_at` wird von zwei Migrationen angelegt (`supabase/manual-migrations/20260605000500_profiles_last_seen.sql`, `20260903000000_last_sign_in_and_last_seen.sql`) — beide idempotent.
- Zeitstempel wird **clientseitig** erzeugt (`new Date().toISOString()` in `presence.functions.ts:22`), nicht per `now()` in der Datenbank. Beides ist UTC, ein echtes Zeitzonenproblem besteht nicht — aber eine falsch gestellte Client-Uhr verfälscht den Wert.
- Autorisierung: Server-Function und RPC prüfen ausschließlich Rolle `admin` (`last-sign-ins.functions.ts:36-43`; Migration `20260903000000...sql:13-15`), obwohl `admin_mitarbeiter` in derselben Datei sonst als vollwertiges Admin-Konto gilt (`admin.chat.tsx:203-205`).
- RLS der Tabelle selbst ist unproblematisch (eigenes Profil aktualisierbar, Admins/Admin-Staff lesend berechtigt).
- Fehler werden an drei Stellen stillschweigend verschluckt (fehlende Spalte beim Schreiben und Lesen, `catch {}` im Heartbeat) — Ausfälle sind dadurch unsichtbar.

## 4. Frontend

- `formatLastActive` rechnet bei jedem Render gegen `Date.now()` — der Text altert also, bekommt aber keine neuen Daten.
- `getLastSignIns` wird nur in `loadConversations()` aufgerufen, und das läuft ausschließlich beim Mount (`admin.chat.tsx:160-163`). Realtime und der 25-s-Poll aktualisieren nur Nachrichten/Unread, **nie** die Aktivitätswerte.
- `last_seen_at` fehlt in der Profil-Selektion der Liste (`admin.chat.tsx:173`) und in `list_chat_conversations`; der Wert kommt ausschließlich aus dem Nachlade-Call.

## 5. Konkrete Ursachen

1. **Kein Refetch:** Aktivitätswerte werden pro Sitzung genau einmal geladen und danach eingefroren. Ein den ganzen Tag geöffnetes Admin-Postfach zeigt zwangsläufig veraltete Zeiten.
2. **Falsche Mischquelle:** `latestTimestamp(lastSeenAt, lastSignInAt, lastFromEmployeeAt)` lässt die letzte **Nachricht** und den letzten **Login** als „Aktivität" durchgehen. Ein Mitarbeiter, der vor Wochen zuletzt schrieb, kann aktueller wirken als er ist — bzw. der Wert springt zwischen semantisch verschiedenen Dingen.
3. **Rollenbruch:** Für Rolle `admin_mitarbeiter` wirft `getLastSignIns` „Nicht autorisiert" → `activityError` → alle Kontakte zeigen dauerhaft „Status unbekannt".
4. **Zwei widersprüchliche Status-Systeme:** Admin sieht Heartbeat/Presence, Mitarbeiter sieht das manuelle `leader_online`-Flag.
5. **Stille Ausfälle:** Fehlt die Migration oder ist Realtime nicht verfügbar, liefert das System kommentarlos „Keine Aktivität" statt einer Fehlermeldung; der Presence-Channel fällt bei self-hosted Realtime-Problemen unbemerkt auf leer zurück (`use-presence.ts:72-76`).

**Offen (nur an der Live-DB prüfbar):** ob `profiles.last_seen_at` dort tatsächlich existiert und gefüllt ist, und ob `admin_mitarbeiter`-Konten betroffen sind.

## Empfohlener Fix (auf deine Freigabe)

1. **Verifikation zuerst:** auf der produktiven DB prüfen, ob `last_seen_at` existiert und aktuelle Werte hat (`select count(*), max(last_seen_at) from profiles`), und welche Rolle deine Admin-Konten haben.
2. **Semantik klären:** „Zuletzt aktiv" nur noch aus `last_seen_at` speisen; Login und letzte Nachricht nur im Tooltip als Zusatzinfo, nicht als Aktivitätsquelle.
3. **Regelmäßiges Nachladen:** die Aktivitätswerte alle 60 s (und bei Tab-Fokus) neu abrufen, ohne die ganze Liste neu aufzubauen.
4. **Rollen angleichen:** Server-Function und RPC `get_last_sign_ins` auf `admin` **und** `admin_mitarbeiter` öffnen (neue idempotente Migration in `supabase/manual-migrations/`).
5. **Zeitstempel serverseitig:** `last_seen_at` per `now()` in der Datenbank setzen statt Client-Uhr.
6. **Fehler sichtbar machen:** fehlende Spalte / Presence-Ausfall als klaren Hinweis in der Admin-Oberfläche zeigen statt still „Keine Aktivität".
7. **Optional:** `leader_online` entweder klar als „manueller Status" beschriften oder durch echte Presence ersetzen — deine Entscheidung.
