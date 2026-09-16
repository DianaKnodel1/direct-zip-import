# Lead-Ereignis direkt beim Formular-Absenden feuern

## Befund (live geprüft)

- `https://cr-beratung.solutions/danke` enthält den Pixel korrekt: ID `2113512246264776`, `FIRE_LEAD=true`, `fbq('track','Lead')` ist im Seiten-Code vorhanden.
- Dein Seller erwartet das Lead-Ereignis aber **direkt nach dem Absenden auf `/bewerben`** (beim „Fast geschafft!"-Moment) — so wie er es in seiner Nachricht beschreibt.
- Aktuell feuert Lead erst nach der Weiterleitung auf `/danke`. Geht dabei etwas schief (Besucher bricht ab, Weiterleitung blockiert, Consent nicht erteilt), verliert Meta die Conversion.

## Was gemacht wird

1. **Lead direkt beim Absenden:** In `src/landing-themes/_shared/form-section.js` wird nach erfolgreichem Submit (vor der Weiterleitung auf `/danke`) zusätzlich `fbq('track','Lead')` gefeuert — aber nur, wenn das Pixel geladen ist (Consent erteilt bzw. nicht nötig). Doppelzählung wird vermieden: Auf `/danke` feuert Lead dann nicht noch einmal, wenn es schon beim Absenden gesendet wurde (Kennzeichen in der Weiterleitungs-URL, z. B. `lead=1`).
2. **`buildPixelBlock` in `landing-server/server.ts` + `server.js`:** Auf `/danke` wird `FIRE_LEAD` nur dann aktiv, wenn die URL kein `lead=1` trägt — so bleibt es bei genau einem Lead pro Bewerbung, egal welcher Weg greift.
3. **Robustheit:** Schlägt das Feuern beim Absenden fehl (Pixel blockiert), greift weiterhin das Lead auf `/danke` als Rückfallebene.

## Technische Details

- `form-section.js`: nach `setStatus('success')` / vor `location.assign('/danke'+qs)` — `try{ if(window.fbq) fbq('track','Lead'); }catch(_){}` und `lead=1` an `qs` anhängen.
- `server.ts`/`server.js` `buildPixelBlock(row, mode)`: bei `mode==='danke'` prüfen, ob die Anfrage-URL `lead=1` enthält (Server-seitig über die Request-URL), dann `lead=false`.
- Danach Theme-Assets neu bauen (`scripts/build-theme-assets.mjs`) und deployen (`git pull && bash scripts/deploy.sh` auf dem Portal-Server), inkl. Theme-Abgleich, damit das neue Formular-Skript live geht.

## Prüfung

- Testbewerbung auf `cr-beratung.solutions/bewerben` mit Consent „Einverstanden": Im Netzwerk muss `facebook.com/tr` mit `ev=Lead` **schon vor** dem Wechsel auf `/danke` erscheinen.
- Auf `/danke` darf kein zweites Lead feuern (kein Doppel-Count im Events Manager).
- Ohne Consent: kein Pixel, kein Lead — unverändert.
