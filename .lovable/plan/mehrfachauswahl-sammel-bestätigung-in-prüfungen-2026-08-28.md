# Mehrfachauswahl & Sammel-Bestätigung in „Prüfungen"

Ziel: In der Prüfungsliste mehrere Einreichungen per Checkbox auswählen und in einem Schritt genehmigen (optional auch ablehnen).

## Was der Admin sieht

- Neue Checkbox-Spalte ganz links in der Tabelle, plus Checkbox im Tabellenkopf für „alle sichtbaren auswählen" (respektiert den aktuellen Status-Filter).
- Sobald mindestens eine Zeile ausgewählt ist, erscheint über der Tabelle eine Aktionsleiste: „X ausgewählt" + Button „Genehmigen" + Button „Ablehnen" + „Auswahl aufheben".
- Sicherheitsabfrage vor der Sammelaktion („X Einreichungen genehmigen?"), optionales gemeinsames Kommentarfeld.
- Danach Toast mit Ergebnis („X genehmigt", bei Teilfehlern „X genehmigt, Y fehlgeschlagen") und Neuladen der Liste.
- Klick auf die Checkbox öffnet nicht den Prüf-Dialog; Klick auf die Zeile funktioniert unverändert.

## Regeln

- Auswählbar sind nur Einreichungen mit Status „Eingereicht", „In Prüfung" oder „Nachbesserung" – bereits genehmigte/abgelehnte Zeilen bekommen eine deaktivierte Checkbox.
- Die Auswahl wird zurückgesetzt, wenn der Status-Filter geändert wird oder die Daten neu geladen werden.
- Die Sammel-Genehmigung nutzt exakt dieselbe Logik wie die Einzelprüfung: Status auf `genehmigt` setzen und – falls die Aufgabenvorlage eine Vergütung > 0 hat – eine Transaktion mit dem Vergütungsbetrag anlegen. Keine doppelten Gutschriften bei bereits genehmigten Einträgen.

## Technisch

- Datei: `src/routes/admin.reviews.tsx` (einzige geänderte Datei).
- Neuer State `selectedIds: Set<string>`; abgeleitete Liste `selectableIds` aus `reviewable`.
- Neue Funktion `bulkDecision(decision)`: iteriert über die ausgewählten Assignments, führt pro Eintrag das bestehende Update (`task_assignments.status`, `admin_comment`) aus und legt bei `genehmigt` die `user_transactions`-Zeile analog zu `reviewDecision` an. Fehler werden gezählt statt abgebrochen; am Ende einmal `loadData()`.
- Checkbox-Komponente aus `@/components/ui/checkbox`, Bestätigung über den vorhandenen Dialog-Baustein.
- Keine Änderungen an Datenbank, RLS, Mandantentrennung oder der bestehenden Einzelprüfung.

## Verifizierung

Build und Typecheck; manuelle Prüfung: Einzelauswahl, „Alle auswählen" mit aktivem Filter, Sammel-Genehmigung mit und ohne Vergütung, Verhalten bei 0 Auswahl.
