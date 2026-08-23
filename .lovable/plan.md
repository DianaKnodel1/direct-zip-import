# Fehlende Theme-Bilder ergänzen

## Diagnose (verifiziert)

Skript hat alle Themes mit Bild-Slots in `meta.json` gegen die tatsächlich vorhandenen Dateien im jeweiligen `assets/`-Ordner geprüft. Fehlend:

| Theme | Fehlende Dateien |
|---|---|
| `theme-slate-premium` („Slate", vom User als „State" gemeint) | `hero.jpg`, `about.jpg`, `company.jpg` |
| `theme-emerald-talent` | `hero.jpg`, `about.jpg`, `company.jpg` |
| `theme-sapphire-matching` | `hero.jpg`, `about.jpg`, `company.jpg` |
| `theme-amber-consult` | `hero.jpg` |
| `theme-10` | `logo.png`, `favicon.png` |

Alle übrigen Themes mit Bild-Slots (ruby-broker, noir-executive, career-atlas, connect-people, device-stack, quality-report, talent-hub, tester-lab, qa-grid, azb-replica, eilers-replica, mirror-site) sind vollständig.

Der Effekt ist derselbe wie zuvor bei Ruby: der Landing-Renderer findet die Datei nicht → im Frontend erscheint nur der Alt-Text („Beratungsgespräch", „Team im Gespräch", „Partnerunternehmen").

## Fix

1. Für jedes betroffene Theme die fehlenden Bilder generieren und in `src/landing-themes/<theme>/assets/` ablegen, jeweils passend zum Look (Slate = neutral/kühl, Emerald = grün/frisch, Sapphire = blau/klar, Amber = warm/gold, Theme-10 = generisches Beratungslogo + Favicon).
   - `hero.jpg` — Beratungsgespräch, hell, modern (~1600×1000)
   - `about.jpg` — Team im Gespräch, freundlich
   - `company.jpg` — Partnerunternehmen / Büro-/Team-Szene
   - `theme-10/logo.png` — schlichtes Wortmarken-Logo, transparent
   - `theme-10/favicon.png` — 512×512 Icon
2. `node scripts/build-theme-assets.mjs` ausführen, damit die Base64-Map `src/lib/theme-assets.generated.ts` neu gebaut wird (sonst zieht der Cloudflare-Worker die Dateien zur Laufzeit nicht).

## Technisches

- Bildquelle: `imagegen--generate_image` (fast) für Fotos als `.jpg`; Logo/Favicon transparent als `.png`.
- Keine Änderungen an `template.html`, `meta.json` oder Renderer — nur Dateien anlegen und Asset-Map neu bauen.
- Bereits gespeicherte Landing-Pages greifen automatisch, sobald der Landing-Server das neue Bundle bekommt.
