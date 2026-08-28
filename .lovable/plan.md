# Diagnose: Chat-Badge zeigt „5", obwohl nichts ungelesen ist

Keine Änderungen vorgenommen. Reine Analyse.

## Kurzfassung der Ursache

Der Badge zählt **mehr Nachrichten als die Chat-Oberfläche überhaupt anzeigt**. Interne System-/KI-Eskalations-Nachrichten (und Nachrichten von anderen Admin-Konten) werden in allen Listen ausgeblendet, aber im Badge mitgezählt. Da sie nie in einem Gespräch geöffnet werden können, werden sie auch nie auf `read = true` gesetzt — die Zahl bleibt dauerhaft stehen (z. B. „5").

## 1./2./3. Wo die Zahl berechnet wird

Es gibt **kein React Query und keine gemeinsame Quelle**. Vier unabhängige Zähler:

| Ort | Abfrage |
|---|---|
| `src/hooks/use-admin-badges.ts:21-25` (Sidebar/Header-Badge) | `chat_messages`, `count exact`, `receiver_id = user.id AND read = false` |
| `src/components/FloatingChat.tsx:124-130` (schwebendes Widget) | identisch |
| `src/routes/admin.chat.tsx:144-148` (`totalUnread`, Summe der Gesprächs-Badges) | aus RPC `list_chat_conversations` |
| RPC `list_chat_conversations` (`supabase/manual-migrations/20260901000000_...sql:22-52`) | `COUNT(*) FILTER (WHERE sender_id = partner_id AND read = false)`, **mit** Ausschluss von `🤖 KI-Eskalation%`, `[ESCALATE]%` und Admin-Partnern |
| `NotificationBell.tsx:40` | andere Tabelle (`notifications`), unabhängig |

Tabelle: `public.chat_messages`, Spalten `receiver_id`, `sender_id`, `read` (boolean, kein `read_at`), `message`, `is_system` (existiert, wird von den Zählern **nicht** genutzt).

## 4./5. Empfänger-Trennung und eigene Nachrichten

Beides korrekt: `receiver_id = user.id` schließt selbst gesendete Nachrichten automatisch aus; die RPC zählt nur `sender_id = partner_id`.

## 6. Mark-as-read

- Mitarbeiter: `src/routes/_employee/chat.tsx:128-132` setzt beim Laden **alle** eigenen ungelesenen Nachrichten auf `read = true` — vollständig.
- Admin: `src/routes/admin.chat.tsx:360-362` setzt nur `sender_id = <geöffneter Mitarbeiter> AND read = false`. **Nachrichten, die zu keinem sichtbaren Gespräch gehören (Systemnotizen, Admin-zu-Admin), können nie gelesen markiert werden.**

## 7. Cache / Polling

Kein React Query, kein staleTime. `use-admin-badges.ts:45-57` lädt neu per Realtime-Subscription auf `chat_messages` plus `setInterval(load, 60_000)`. Ein rein „veralteter Cache" scheidet damit als Ursache aus — die 5 ist **real in der Datenbank vorhanden**, nur nicht sichtbar/markierbar.

## 8. Gleiche Quelle?

Nein. Sidebar-Badge (`use-admin-badges`) und Chat-Seite (`totalUnread` aus RPC) rechnen unterschiedlich — genau daher die Diskrepanz „Badge 5 / Chatliste 0". Zusätzlich hat `FloatingChat` einen dritten eigenen Zähler.

## 9. Gegenüberstellung Testperson

In dieser Umgebung steht kein SQL-Zugriff auf das selbst gehostete Backend zur Verfügung, deshalb noch nicht gegen echte Daten geprüft. Verifizierender Query (nur lesend, von dir auszuführen):

```sql
select id, sender_id, left(message, 40) as msg, is_system, created_at
from public.chat_messages
where receiver_id = '<USER_ID>' and read = false
order by created_at desc;
```

Erwartung laut Analyse: 5 Zeilen, alle mit `[ESCALATE]`/`🤖 KI-Eskalation`-Präfix oder von einem Admin-Konto als Absender.

## 10. Ähnliches Risiko bei anderen Kennzahlen

Ja, zwei Muster:

- **Definitions-Abweichung:** `use-admin-badges.ts:29` zählt KYC nur `status = 'eingereicht'`, `admin.index.tsx:24` dagegen `('eingereicht','in_pruefung')`. Ebenso Bewerbungen: `'neu'` vs. `('neu','eingegangen')`.
- **Kein Invalidieren:** `AdminDataContext` (`:253-255`) lädt einmalig beim Mount, ohne Polling/Realtime — Dashboard-Kacheln bleiben nach Aktionen bis zum Reload stehen.

## Empfohlene minimale Korrektur (noch nicht umgesetzt)

Ein Eingriff, eine Datei: in `src/hooks/use-admin-badges.ts` die Chat-Zählung an die Logik der Gesprächsliste angleichen — Systemnotizen ausschließen, z. B. per `.eq("is_system", false)` bzw. `.not("message","like","[ESCALATE]%")` etc. Optional identisch in `FloatingChat.tsx:124-130`.

Alternative (sauberer, aber DB-Änderung): eine RPC `count_unread_chat()` als einzige Quelle für alle Badges.

Nach Freigabe setze ich nur die Variante um, die du wählst.
