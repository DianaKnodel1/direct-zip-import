# Meta-Pixel-ID pro Landing Page jederzeit selbst ändern

## Befund (geprüft)

Das Feld ist bereits vorhanden und live: Im **Landing-Generator** gibt es unter **Branding** das Feld **Facebook-Pixel-ID** (`meta_pixel_id`), pro Landing Page. Der Landing-Server baut daraus bei jedem Seitenaufruf den vollständigen Pixel-Code (init + PageView auf allen Seiten, Lead auf `/danke`, Consent-Gate in DE/EU).

Es ist **keine Codeänderung und kein Deploy nötig**.

## Was du tun kannst (jederzeit, pro Landing Page)

1. Portal öffnen → **Admin → Landing-Generator**.
2. Die gewünschte Landing Page auswählen (z. B. app24-gmbh.com).
3. Unter **Branding → Facebook-Pixel-ID** die neue ID eintragen — z. B. `2113512246264776`.
4. **Speichern.**

## Wirkung

- Nach ca. 1 Minute (60-Sekunden-Zwischenspeicher des Landing-Servers) lädt die Seite den Pixel mit der neuen ID.
- Nur die Zahl ändern — der restliche Pixel-Code (Skript, PageView, Lead-Event, Einwilligungs-Hinweis) bleibt automatisch gleich und korrekt.
- Feld leeren = Pixel wird komplett entfernt (kein Tracking, kein Consent-Hinweis).
- Jede Landing Page / Domain hat ihre eigene ID — nie feuert ein fremdes Pixel auf einer fremden Domain.

## Hinweis zu „der Code ändert sich immer"

Der eigentliche Pixel-Code von Meta ist immer gleich — nur die **ID** (die lange Zahl) ist pro Werbekonto verschieden. Genau diese ID kannst du über das Feld jederzeit austauschen.

## Prüfung nach dem Eintragen

1. Landing Page öffnen, Consent „Einverstanden" klicken.
2. Im Browser (Entwicklerwerkzeuge → Netzwerk) Anfrage an `facebook.com/tr` mit der neuen ID und `ev=PageView` sehen.
3. Testbewerbung → auf `/danke` muss `ev=Lead` mit derselben ID erscheinen.
4. Im Meta Events Manager kontrollieren, dass PageView und Lead ankommen.
