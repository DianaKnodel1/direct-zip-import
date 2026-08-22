# Projekt-Aufräumung

Befund: 319 TS/TSX-Dateien (65k Zeilen), dazu 48 Shell-Skripte, 28 Landing-Themes (7,3 MB), tote Logos und deaktivierte Features. Aufgeteilt nach Risiko — Phase 1 sofort sicher, Phasen 2–4 nur nach deiner Freigabe pro Block.

## Phase 1 — Sofort sicher (keine Funktion ändert sich)

1. **Tote Logo-SVGs im Projektroot löschen** — 9 von 11 werden nirgends referenziert (AOK, BBVA, DKB, Deutsche Post, HypoVereinsbank, SAP, WebID, Allianz, Deutsche Bank). Commerzbank + Debeka bleiben (in Benutzung).
2. **Deaktivierten Staff-Admin-Code entfernen** — `AdminLayout.tsx`: `STAFF_ALLOWED_PREFIXES`, `STAFF_HOME`, `staffNavGroups` sind leer/deaktiviert; der zugehörige Redirect-Block ist tot.
3. **`public/theme-preview/` löschen** (815 KB) — alte Theme-Vorschau-HTML, wird von der App nicht geladen.
4. **Diagnose-Reste** — bereits erledigt: Diagnose-Box aus Bewerbungen entfernt inkl. ungenutztem `phaseReason`.

## Phase 2 — Code-Struktur (risikoarm, macht Wartung leichter)

5. **Riesen-Dateien aufteilen**: `admin.landing-generator.tsx` (1737 Zeilen), `admin.chat.tsx` (1329), `admin.tenants.tsx` (813) → Unterkomponenten in `src/components/admin/`.
6. **Ungenutzte UI-Komponenten & Hooks entfernen** — Abgleich aller `src/components/ui/*` und `src/hooks/*` gegen tatsächliche Importe; Unreferenziertes löschen.
7. **Tote Routen prüfen** — z. B. `admin.post.tsx`, `tts-test.tsx`, `interview-test.tsx`, `admin.webid-sim.tsx`: Test-/Demo-Seiten, wenn nicht mehr gebraucht → raus.

## Phase 3 — Skripte & Server-Verzeichnisse (nur mit deiner Info)

8. **`scripts/` ausmisten** (48 Dateien): viele einmalige Diagnose-Skripte (`diagnose-*.sh`, `fix-*.sh`, `check-*.sh`) und 5 Generations-Migrate-Skripte (`migrate-to-backend*.sh`). Was davon läuft noch per Cron auf deinem Server? Der Rest fliegt raus. `scripts/email-test/` komplett?
9. **`bot-runner/`, `webid-sim-server/`** — laufen die noch produktiv? Wenn nein: löschen.
10. **`landing-server/`**: `server.js` (32 KB) und `server.ts` (18 KB) existieren parallel — `package.json` startet nur `server.js`. `server.ts` ist vermutlich die alte Version → löschen, wenn du das bestätigst.

## Phase 4 — Landing-Themes (nur mit DB-Einsicht)

11. **28 Themes, nur ~10 haben `meta.json`** (also fertig registriert). Themes ohne `meta.json` sind evtl. Altlasten. Aber: Aktive Tenants in der Datenbank referenzieren Themes per Slug — bevor ich lösche, brauche ich die Liste der tatsächlich genutzten Themes (SQL im Supabase-Editor: `select distinct landing_theme from tenants;` o. ä.) oder du sagst mir, welche Domains live sind. Danach: ungenutzte Themes + ihre `_shared/form-section-*`-Dateien entfernen und `theme-assets.generated.ts` neu bauen.

## Was bewusst bleibt

- `src/integrations/supabase/types.ts` (3,8k Zeilen) und `routeTree.gen.ts` — generiert, kein Müll.
- `supabase/manual-migrations/` — historische Migrationen, wichtig für Nachvollziehbarkeit.
- `docs/`, `RUNBOOK.md`, `AUDIT.md` — Doku, außer du willst sie kürzen.

## Ablauf

Phase 1 setze ich direkt um. Für Phasen 3–4 brauche ich kurze Antworten (siehe Fragen im Chat). Phase 2 mache ich auf Wunsch direkt im Anschluss.
