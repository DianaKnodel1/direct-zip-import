# Warum noch „/bewerben" — und wie die Danke-Seite scharf wird

## Befund (geprüft im Code)

Nein, das ist nicht normal — der Screenshot zeigt weiterhin das alte Formular-Skript
(Pop-up statt Weiterleitung, Schaltfläche „Jetzt bewerben →").

Grund: Der Landing-Server lädt die Theme-Dateien (darin steckt das Formular-Skript)
**nur beim Start von der eigenen Festplatte** und holt neue Dateien ausschließlich,
wenn das Portal im Minuten-Signal „Abgleich nötig" meldet. Dieses Kennzeichen wird
beim Deploy nur gesetzt, wenn die Zugangsdaten zur Datenbank hinterlegt sind — beim
letzten Deploy erschien genau deshalb die Warnung „Kein TARGET_DB_URL".
Der Deploy hat also nur die Server-Dateien erneuert, nicht das Formular-Skript.

Ein einfacher Neustart des Landing-Servers hilft nicht: Er lädt dann wieder die
alten Dateien von der Platte. Es muss erst ein Abgleich ausgelöst werden.

## Lösung (drei Wege, einer reicht)

**A — Im Portal (am einfachsten):**
Infrastruktur-Seite öffnen, beim Landing-Server „Themes neu synchronisieren"
auslösen. Innerhalb einer Minute holt der Server die neuen Dateien und startet neu.

**B — Direkt auf dem Landing-Server:** die Theme-Dateien einmalig von Hand holen
und anschließend den Dienst neu starten (Befehle liefere ich nach Freigabe).

**C — Dauerhafte Lösung (empfohlen, zusätzlich):**
In `/opt/apps/portal/.env.server` die Zeile `TARGET_DB_URL=...` ergänzen.
Danach löst **jeder** Deploy den Theme-Abgleich automatisch aus und dieser
Zwischenschritt entfällt für immer.

## Danach prüfen

1. Browser-Zwischenspeicher umgehen (Seite mit Strg+F5 neu laden) —
   das Skript wird 5 Minuten lang zwischengespeichert.
2. Testbewerbung über `/bewerben` abschicken.
3. Erwartung: Die Adresse wechselt auf `/danke`, dort steht der Danke-Text und
   die grüne Schaltfläche heißt **„Jetzt Termin vereinbaren →"**.
4. Erst dann `/danke` bei Meta als Lead-Ziel hinterlegen.

## Technische Details

- `landing-server/server.ts` liest `themes/<id>/{template.html,style.css,script.js}`
  einmalig beim Prozessstart von der lokalen Platte.
- `landing-server/heartbeat.sh → resync_themes()` lädt diese Dateien von
  `/api/public/landing-server-files/...`, aber nur bei `resync_needed: true`.
- `resync_needed` hängt an `landing_servers.themes_resync_requested_at`;
  gesetzt wird die Spalte von `scripts/sync-landing-server.sh` (nur mit
  `TARGET_DB_URL`) oder über die Admin-Funktion in
  `src/lib/landing-servers.functions.ts`.
- Keine Codeänderung nötig: `/danke`, die Weiterleitung und die neue Beschriftung
  sind im Repo bereits vorhanden und ausgeliefert.
