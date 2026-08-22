# Chat stabilisieren, „Aktiv vor…" korrigieren, Recorder-Ablauf

## 1. „Aktiv vor …" zeigt das Falsche

**Befund (geprüft in `src/routes/admin.chat.tsx`, Zeile 1090):** Angezeigt wird `lastSeenAt ?? lastSignInAt`.
- `last_seen_at` ist der Heartbeat aus `src/hooks/use-presence.ts` — er läuft **nur**, solange der Mitarbeiter das Portal geöffnet hat, und nur, wenn die Migration `20260903000000_last_sign_in_and_last_seen.sql` auf der Datenbank eingespielt ist. Fehlt die Spalte, wird still auf den letzten Login zurückgefallen — und der kann Tage alt sein, obwohl der Mitarbeiter gestern geschrieben hat.
- Dass der Mitarbeiter **eine Nachricht geschrieben** hat, fließt heute gar nicht in den Status ein. Genau das ist dein Fall (Marcel).

**Fix:**
- Aktivitätszeit = der **jüngste** der drei Werte: Heartbeat (`last_seen_at`), letzter Login (`last_sign_in_at`) und **letzte eigene Nachricht des Mitarbeiters** (`lastFromEmployeeAt`, liegt in der Chatliste bereits vor).
- Tooltip zeigt die Quellen einzeln: „Zuletzt im Portal / Letzter Login / Letzte Nachricht", damit du siehst, woher der Wert stammt.
- „Status unbekannt" nur noch, wenn wirklich alle drei Quellen leer sind.
- Prüfen (und ggf. nachziehen), ob `profiles.last_seen_at` und die RPC `get_last_sign_ins` auf der Live-Datenbank existieren; sonst bleibt der Heartbeat wirkungslos.

## 2. Chat als echter Live-Chat — Nachrichten dürfen nicht verschwinden

**Befund:** Laden, Senden (optimistisch + Retry) und Realtime-Insert sind vorhanden, es fehlen aber drei Dinge, die genau das Verhalten „Nachricht war da, ist weg" erzeugen:

1. **Kein Resync nach Verbindungsabriss.** Realtime wird nur beim ersten `SUBSCRIBED` nachsynchronisiert. Schläft der Laptop, wechselt das WLAN oder liegt das Handy 20 Minuten in der Tasche, fällt der Socket still aus — neue Nachrichten kommen erst nach Reload. Für den Mitarbeiter sieht es aus, als hätte niemand geantwortet.
2. **Neu laden verwirft ungesendete Nachrichten.** `loadData()` im Mitarbeiter-Chat und `selectConversation()` im Admin-Chat **ersetzen** die Liste. Eine Nachricht mit Status „wird gesendet"/„fehlgeschlagen" verschwindet dabei spurlos — inklusive Retry-Möglichkeit.
3. **Nur die letzten 200 Nachrichten**, „Ältere laden" ist manuell. Bei langen Verläufen wirkt der Anfang „gelöscht".

**Umsetzung (`src/routes/_employee/chat.tsx`, `src/routes/admin.chat.tsx`, `src/components/FloatingChat.tsx`):**
- Gemeinsame Merge-/Sync-Logik in ein Modul `src/lib/chat-sync.ts` ziehen (heute dreimal kopiert) und dort **Pending-/Failed-Nachrichten immer erhalten**: neu geladene Serverdaten überschreiben nur echte IDs, lokale `pending-…`-Einträge bleiben stehen, bis sie gespeichert oder verworfen sind.
- **Resync-Trigger** in allen drei Oberflächen: bei `visibilitychange` (Tab wieder sichtbar), `online`-Event, Realtime-Status `CHANNEL_ERROR`/`TIMED_OUT`/`CLOSED` (mit Reconnect) und zusätzlich alle 25 s als stiller Fallback-Poll. Der Poll holt nur Nachrichten neuer als die letzte bekannte — minimale Last.
- **Verbindungsanzeige** im Chatkopf: „Live" bzw. „Verbindung unterbrochen – wird neu verbunden", damit sichtbar ist, wenn gerade nichts ankommt.
- **Ausstehende Sendungen überleben den Seitenwechsel**: `pending`/`failed`-Nachrichten pro Gesprächspartner in `sessionStorage` sichern und beim Öffnen wieder einblenden (mit „Erneut senden").
- **Automatisch ältere Nachrichten nachladen**, wenn nach oben gescrollt wird, statt nur per Button.
- Keine Änderung an Rechten, RLS oder am Speicherformat — nur Anzeige- und Sync-Verhalten.

## 3. Bot-Aufnahme: Ablauf + Korrektur

**Befund (`src/routes/api/public/bot-recorder-script.ts`):** Das Recorder-Skript lebt in der geöffneten Seite. Tab wechseln, minimieren, zurückkommen — alles unproblematisch (Schritte liegen in `sessionStorage`, Upload alle 3 s). **Aber:** bei einem echten Seitenwechsel (neue URL, neuer Tab, Login-Redirect der Bank) ist das Skript weg und die Aufnahme läuft stumm weiter ins Leere, ohne Warnung.

**Fix:** Beim Start ein Flag in `sessionStorage` setzen und die Statusleiste nach einem Seitenwechsel deutlich machen: erscheint sie nicht, muss das Lesezeichen erneut geklickt werden (die bisherigen Schritte gehen dabei nicht verloren, sie werden fortgesetzt). Zusätzlich im Panel eine kurze Ablaufhilfe mit genau diesem Hinweis.

**Der komplette Ablauf, den du machen musst:**
1. Admin → Bots → „Neue Aufnahme": Name + Start-URL eintragen, „Aufnahme starten".
2. Den angezeigten Lesezeichen-Link **einmal** als Lesezeichen in die Leiste ziehen („Bot-Aufnahme").
3. Bankseite öffnen → Lesezeichen anklicken → unten rechts erscheint „Aufnahme läuft · N Schritte".
4. Den Antrag ganz normal durchklicken. Passwörter/Werte werden **nicht** übertragen, nur Feldnamen.
5. **Nach jedem Sprung auf eine andere Seite/Domain** prüfen, ob die Leiste noch da ist — wenn nicht: Lesezeichen erneut klicken.
6. Am Ende „Stopp" (oder im Panel „Beenden"). Steht dort „Senden blockiert", stattdessen „Kopieren" und im Panel einfügen.
7. Im Panel „Ablauf erzeugen" → Vorschau prüfen: Platzhalter-Übersicht zeigt alle `{{felder}}`; rot markierte fehlen im Mitarbeiter-Profil und müssen dort gepflegt (oder im Schritt geändert) werden.
8. „Als Bot-Profil speichern" → Bot testweise auf einem Auftrag laufen lassen.

## 4. Vor dem Deploy

- Build-Log prüfen (`build OK`), Lint/Typecheck laufen lassen.
- Chat-Test im Preview: Nachricht senden, Tab wechseln, Netz kurz trennen → Nachricht bleibt sichtbar, Resync greift, nichts verschwindet.
- Kontrolle, ob die offenen Migrationen (`20260903`, `20260904`, `20260905`) auf der Live-Datenbank eingespielt sind — sonst bleibt der Aktiv-Status ungenau.
