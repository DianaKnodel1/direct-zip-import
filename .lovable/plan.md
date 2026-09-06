# Landing-Generator: „Seite duplizieren" statt Neu-Erstellen

## Ziel
Bestehende Landing Page mit einem Klick komplett kopieren (Theme, Branding, alle Texte/Slots, Einstellungen) und als neue, editierbare Seite öffnen — du änderst dann nur noch, was anders sein soll. Kein neuer Generator, kein Umbau des Bestehenden.

## Was gebaut wird

### 1. „Duplizieren"-Button in der Landing-Liste
- In `src/routes/admin.landing-generator.tsx` bekommt jede gespeicherte Landing in der Liste neben „Bearbeiten" einen **„Kopieren"**-Button.
- Klick → lädt die komplette Landing (Theme, Branding, Slot-Texte, Logo, Flow-Typ, Interview-/Booking-Einstellungen) in den Editor, **ohne** sie zu überschreiben:
  - `editingId` wird geleert → Speichern erzeugt eine **neue** Seite
  - Firmenname bekommt „ (Kopie)" angehängt
  - Domain, Slug und Tenant-ID werden **geleert** (müssen bewusst neu gesetzt werden, damit nicht zwei Seiten auf derselben Domain/demselben Mandanten landen)
- Alles andere bleibt 1:1 erhalten.

### 2. Vorbelegung aus bestehender Seite beim „Neu"
- Beim Klick auf „Neue Landing" erscheint künftig optional eine Auswahl: **„Leer starten"** oder **„Von Vorlage übernehmen"** (Dropdown mit allen bestehenden Landings) — gleiche Logik wie Duplizieren.

### 3. Nicht kopiert werden (Sicherheit)
- Domain/Slug (darf nicht doppelt vergeben sein)
- Tenant-ID, Partner-Verknüpfung, Calendly-URL der verknüpften Fast-Track-Seite — werden geleert, damit keine Bewerbung versehentlich beim falschen Mandanten landet.

## Bewusst NICHT im Plan
- Kein neuer Generator-Assistent/Wizard (der aktuelle Editor bleibt)
- Keine Änderung an Themes, Landing-Server oder DNS-Logik

## Technische Details
- Änderungen nur in `src/routes/admin.landing-generator.tsx`: neuer `handleDuplicateLanding(id)` (baut auf `getLandingPage` + `normalizeSlotsForTheme` auf, analog `handleEditLanding`, aber ohne `editingId` und mit bereinigten Feldern), plus Button in der Liste und Vorlagen-Auswahl beim Neu-Dialog.
- Keine Datenbank-Migration nötig.
