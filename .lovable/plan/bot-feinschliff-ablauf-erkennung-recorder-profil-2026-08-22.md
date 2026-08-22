# Bot-Feinschliff: Ablauf-Erkennung + Recorder → Profil

## Ziel

Der Bot-Runner soll nicht mehr blind eine Schrittliste abarbeiten, sondern **erkennen, auf welcher Seite des Ablaufs er sich befindet**, und entsprechend reagieren (weitermachen, springen, an Admin übergeben). Zusätzlich wird die Umwandlung **Recorder-Aufnahme → fertiges Bot-Profil** verbessert, damit aufgenommene Bank-/Anbieter-Registrierungen ohne Nacharbeit laufen.

Der Bot-Runner-Dienst läuft bereits auf dem Portal-Server. Ziel-Strecke: Bank-/Anbieter-Registrierung.

---

## Teil 1: Seiten-/Ablauf-Erkennung im Runner (`bot-runner/server.ts`)

1. **Seitenzustand-Erkennung (`detectPageState`)**
   Neue Funktion, die die aktuelle Seite anhand von URL und sichtbarem Text klassifiziert:
   - `captcha` / Bot-Sperre → **sofort** Übergabe an Admin mit klarem Grund (heute: der Bot wartet minutenlang auf Elemente, die nie kommen)
   - `maintenance` / Fehlerseite → Abbruch mit verständlicher Meldung (teilweise vorhanden, wird erweitert)
   - `legitimation` (VideoIdent/PostIdent/photoTAN) → automatischer Handoff, auch wenn das Profil keinen `handoff`-Schritt hat
   - `confirm` (Vorgangsnummer/Antragsnummer sichtbar) → automatisch Nummer auslesen, auch wenn kein `extract`-Schritt im Profil steht
   - `login` (Passwort-Login statt Registrierungsformular) → Handoff „Zugang erwartet"

2. **Schritt-Synchronisation (`on_page`-Marker)**
   - Schritte bekommen optional einen Marker (`on_page`: URL-Muster oder Text), der beschreibt, auf welcher Seite sie ausführbar sind.
   - Vor jedem Schritt prüft der Runner, ob die Seite passt. Wenn nicht, sucht er im Profil vorwärts nach dem passenden Schritt (z. B. weil die Bank einen Zwischenschritt übersprungen hat) statt zu scheitern.
   - Ergebnis: Der Bot „findet sich im Ablauf wieder", statt beim kleinsten Abweichen stecken zu bleiben.

3. **`advance` robuster**
   - Mehr Weiter-Button-Texte (z. B. „Jetzt starten", „Zur Kontoeröffnung", „Antrag weiterführen")
   - Stagnations-Erkennung: gleiche URL nach Klick → nicht endlos weiterklicken
   - Stopp bei Captcha/Bot-Sperre

4. **Eingabe-Feinschliff (`fill`)**
   - Datumsfelder: Format der Seite erkennen (TT.MM.JJJJ vs. JJJJ-MM-TT anhand von Placeholder/`inputmode`) und Wert passend umwandeln
   - Checkboxen/Radio: echtes Anhaken statt Blind-Klick
   - IBAN/BLZ: Leerzeichen-Formatierung tolerant

## Teil 2: Recorder → Profil (`src/lib/recording-clean.ts`, `BotRecorderPanel.tsx`)

1. **Bessere Bereinigung der Aufnahme**
   - Lange Pausen zwischen zwei Aktionen (> 3 s) erzeugen automatisch `wait_for`-Schritte (Seitenladezeit), statt dass der Bot später in Timeouts läuft
   - Absenden/Vorgangsnummer im Ablauf erkannt → automatisch `advance`- und `extract`-Bausteine ans Ende
   - `on_page`-Marker aus den aufgezeichneten URLs automatisch an die Schritte schreiben (nutzt Teil 1)
   - Passwort-Felder: Platzhalter `{{password}}` aus den Zugangsdaten, niemals Klartext

2. **Vorschau vor dem Speichern editierbar**
   - Im Recorder-Panel werden die bereinigten Schritte als bearbeitbares JSON angezeigt (wie im Profil-Editor), bevor sie als Profil gespeichert werden
   - Platzhalter-Übersicht: Liste aller `{{platzhalter}}` mit Hinweis, welche Felder der Mitarbeiter im Profil braucht — fehlende Standardfelder (E-Mail, Vorname …) werden markiert

## Teil 3: Verifikation

- Unit-Test der Bereinigungslogik mit Beispiel-Mitschnitten (Doppelklicks, Cookie-Banner, Pausen, Datumsfeld)
- Build-Check des Portals; Syntax-Check des Runners (`bun --check` bzw. Startprobe)
- Hinweis zum Server: nach dem Update `bot-runner/` neu auf den Server spielen und `systemctl restart bot-runner`

---

## Technische Details

- **Runner:** `bot-runner/server.ts` (728 Zeilen, Schritt-DSL: goto/fill/click/select/wait/wait_for/screenshot/advance/extract/prompt/handoff). Neue Funktionen `detectPageState()`, `syncToStep()`; Erweiterung des `Step`-Interfaces um `on_page`.
- **Bereinigung:** `src/lib/recording-clean.ts` (`cleanRecording`, `pickSelectors`, `guessPlaceholder`) — reine Funktion, gut testbar.
- **UI:** `src/components/admin/BotRecorderPanel.tsx` (Vorschau-Dialog um JSON-Editor + Platzhalter-Liste erweitern).
- **Keine DB-Änderungen nötig** — `bot_runs`, `bot_profiles`, `bot_recordings` reichen aus; `on_page` lebt im `steps`-JSON des Profils.
- **Bestehende Profile bleiben lauffähig** — `on_page` ist optional; Profile ohne Marker verhalten sich wie bisher.
- **Nicht Teil dieses Plans:** Captcha-Lösung, VideoIdent-Automatisierung (bleibt bewusst Handoff), neue Strecken aufnehmen.

## Offene Punkte für dich

- Nach Fertigstellung: Bot-Runner auf dem Server aktualisieren (`bash scripts/deploy.sh` bzw. Runner-Verzeichnis neu übertragen + `systemctl restart bot-runner`).
- Empfohlen: Danach eine echte Strecke einmal mit dem Recorder aufnehmen und einen Testlauf starten.
