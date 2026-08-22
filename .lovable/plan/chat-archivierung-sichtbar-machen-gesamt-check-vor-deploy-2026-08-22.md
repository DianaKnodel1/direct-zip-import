# Chat-Archivierung sichtbar machen + Gesamt-Check vor Deploy

## Warum du die Mehrfachauswahl nicht siehst

Die Funktion ist vorhanden, aber praktisch unsichtbar: sie steckt hinter einem
kleinen Icon-Knopf (Häkchen-Symbol) rechts neben "Aktiv" / "Ausgeblendet" —
ohne Beschriftung. Außerdem heißt sie im Text "Ausblenden", nicht "Archivieren",
deshalb erkennt man sie auch beim Suchen nicht.

## Was ich ändere

1. **Sichtbarer Auswahl-Modus**
   - Aus dem Icon wird ein beschrifteter Button "Auswählen" über der Chatliste.
   - Im Auswahl-Modus: echte Checkboxen pro Chat, "Alle / Keine",
     Zähler ("3 ausgewählt") und ein deutlicher Button **"Archivieren"**
     (bzw. "Wiederherstellen" in der Archiv-Ansicht).
   - Umbenennung der Ansicht "Ausgeblendet" → **"Archiv"**, damit die Begriffe
     zusammenpassen. Funktion bleibt identisch (nichts wird gelöscht).

2. **Chat-Prüfung**
   - Kontrolle, dass Massen-Archivierung sauber speichert (auch bei Chats, für
     die noch kein Konversations-Datensatz existiert) und die Liste danach
     korrekt zählt.
   - Prüfung, dass beim Archivieren kein Nachrichtenverlust entsteht und der
     offene Chat sauber geschlossen wird.

3. **Aktiv-Status**
   - Erneute Kontrolle der Anzeige (jüngster Wert aus Heartbeat, letztem Login
     und letzter Mitarbeiter-Nachricht) inkl. Tooltip.
   - Wichtig: die Anzeige wird erst live korrekt, wenn die Migrationen
     `20260903`, `20260904`, `20260905` auf der Datenbank gelaufen sind
     (passiert beim nächsten `scripts/deploy.sh`).

4. **Statistik**
   - Durchsicht von `/admin/statistiken`: Zählung nach Bewerbungsdatum vs.
     Termindatum, Zeitraumfilter, Division-durch-Null bei Quoten.

5. **Bot / Recorder – Feinschliff**
   - Aufnahme: klarere Statusleiste und Hinweis nach Seitenwechsel prüfen.
   - Lauf: Kontrolle der Seiten-Erkennung (`on_page`-Marker, Sprung zum
     passenden Schritt), Timeout-/Retry-Werte, IBAN-/Datums-Normalisierung.
   - Verbesserung: bei "Schritt passt nicht zur Seite" wird ein Screenshot
     mitgeschrieben, damit du im Log direkt siehst, wo der Bot hängt.
   - Fehlende Profilfelder werden vor dem Start rot markiert (bereits da) —
     zusätzlich Blockierhinweis, wenn Pflichtfelder leer sind.

## Prüfung vor dem Deploy

- Typecheck + Build.
- Klick-Durchlauf im Admin-Chat: auswählen, archivieren, wiederherstellen,
  Zähler, Chatverlauf.
- Durchsicht der Logs auf Laufzeitfehler.
- Danach: Deploy + Migrationen `20260903` / `20260904` / `20260905`.

## Technische Details

- Betroffen: `src/routes/admin.chat.tsx` (Auswahl-UI, `bulkSetHidden`,
  Bezeichnungen), ggf. `src/lib/chat-sync.ts` (nur lesend geprüft),
  `bot-runner/server.ts` (Screenshot bei Marker-Abweichung),
  `src/components/admin/BotRecorderPanel.tsx` (Pflichtfeld-Hinweis),
  `src/routes/admin.statistiken.tsx` (nur, falls ein Fehler gefunden wird).
- Datenmodell bleibt unverändert: Archivierung = `chat_conversations.admin_hidden_at`.
