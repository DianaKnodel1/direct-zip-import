# Ruby-Theme: fehlende Bilder ergänzen

## Diagnose (verifiziert)
Der Ordner `src/landing-themes/theme-ruby-broker/assets/` existiert nicht. In `meta.json` sind aber drei Bild-Slots mit Default-Pfaden hinterlegt:

- `hero_image` → `assets/hero.jpg`
- `about_image` → `assets/about.jpg` (Alt-Text „Team im Gespräch")
- `company_image` → `assets/company.jpg` (Alt-Text „Partnerunternehmen")

Weil die Dateien fehlen, laden im gerenderten Theme die Bilder nicht — genau wie im Screenshot sichtbar (nur der Alt-Text ist zu sehen). Alle anderen Themes mit Bild-Slots (z. B. `theme-noir-executive`) haben die passenden Dateien im `assets/`-Ordner.

## Fix

1. Ordner `src/landing-themes/theme-ruby-broker/assets/` anlegen.
2. Drei zum Ruby-Look passende Fotos erzeugen (warmes Rot/Neutral, Personalberatungs-Kontext) und dort ablegen:
   - `hero.jpg` — Beratungsgespräch, hell, moderner Look
   - `about.jpg` — Team im Gespräch (freundlich, echt wirkend)
   - `company.jpg` — Partnerunternehmen / Büro-/Team-Szene
3. `node scripts/build-theme-assets.mjs` ausführen, damit `src/lib/theme-assets.generated.ts` die neuen Assets als Base64 in das Worker-Bundle einbettet (wird sonst zur Laufzeit vom Landing-Renderer nicht gefunden).

## Technisches
- Bildquelle: `imagegen--generate_image` (fast), Format `.jpg`, ca. 1600×1000.
- Keine Änderungen an `template.html`, `meta.json` oder Renderer — nur Dateien hinzufügen und Asset-Map neu bauen.
- Bereits gespeicherte Landing-Seiten mit Ruby-Theme greifen automatisch, sobald das Backend/der Landing-Server das neue Bundle bekommt (Themes-Resync-Flag setzen falls nötig).
