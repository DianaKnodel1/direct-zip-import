# Landing-Page-Server absichern (DDoS, IP verstecken, Härtung)

Ziel: Der neue Landing-Server (190.97.165.152) soll nicht mehr direkt aus dem
Internet ansprechbar sein. Angriffe treffen dann Cloudflare, nicht den Server.

```text
Besucher ──► Cloudflare (Proxy, DDoS-Filter, WAF)
                    │  nur Cloudflare-IPs dürfen rein
                    ▼
             Landing-Server (Caddy ► Renderer)
```

## 1. Echte Server-IP verstecken (Cloudflare-Proxy)

- In Cloudflare bei jeder Landing-Domain den A-Record auf 190.97.165.152 setzen
  und die **orange Wolke aktivieren** (Proxied). Damit sieht die Welt nur noch
  Cloudflare-IPs.
- SSL-Modus pro Domain auf **Full (strict)** stellen.
- Wichtig: Sobald der Proxy an ist, kann Caddy keine Zertifikate mehr per
  „on demand" holen. Stattdessen bekommt der Server ein **Cloudflare
  Origin-Zertifikat** (15 Jahre gültig, kostenlos, in Cloudflare unter
  SSL/TLS → Origin Server erzeugen) — genauso wie es beim WebID-Server
  bereits gemacht wurde.
- Alternative, wenn Domains von Kunden kommen und nicht in deinem Cloudflare
  liegen: diese Domains ungeproxyt lassen (dort bleibt die IP sichtbar) oder
  Cloudflare „Custom Hostnames" (SaaS) buchen.

## 2. Firewall: nur noch Cloudflare darf auf 80/443

- ufw so einstellen, dass 80/443 ausschließlich für die offiziellen
  Cloudflare-IP-Bereiche offen sind, alles andere blockiert.
- Ein kleines Skript hält die Cloudflare-Liste täglich aktuell.
- Effekt: Wer die IP kennt, kommt trotzdem nicht mehr direkt an die Seiten.

## 3. SSH absichern

- SSH nur mit Schlüssel (Passwort-Login aus), Root-Login auf „nur Schlüssel".
- SSH-Port bleibt offen, aber am besten nur für deine eigene IP freigeben
  (sonst Port ändern + fail2ban).
- fail2ban installieren (sperrt IPs nach Fehlversuchen).
- Automatische Sicherheitsupdates aktivieren.

## 4. Schutz auf Anwendungsebene

- Cloudflare: „Under Attack"-Modus als Notfallschalter, Bot Fight Mode,
  Rate-Limiting-Regel für das Bewerbungsformular (z. B. 10 Absendungen pro
  Minute und IP).
- Caddy: Verbindungs-/Anfrage-Limits und Timeouts, plus Sicherheits-Header
  (HSTS, X-Frame-Options, Referrer-Policy).
- Echte Besucher-IP korrekt durchreichen (Cloudflare-Header), damit Sperren
  und Logs stimmen und nicht alle Besucher gleich aussehen.

## 5. Betrieb und Kontrolle

- Nur die nötigen Dienste laufen lassen; Renderer bleibt auf 127.0.0.1.
- Backups laufen über den Backup-Server (bereits geplant).
- Prüfung am Ende: direkter Aufruf über die IP muss scheitern, Aufruf über die
  Domain muss funktionieren.

## Was ich dafür im Projekt ändere

- `landing-server/Caddyfile`: Variante für Cloudflare-Betrieb (Origin-Zertifikat
  statt on-demand), Sicherheits-Header, Limits, `trusted_proxies` für
  Cloudflare.
- Neues Skript `scripts/harden-landing-server.sh`: ufw + Cloudflare-IP-Regeln,
  fail2ban, unattended-upgrades, SSH-Härtung — in einem Durchlauf, wiederholbar.
- Neues Skript `scripts/update-cloudflare-ips.sh` + täglicher Timer.
- Kurzanleitung `docs/SERVER-HARDENING.md` mit den Cloudflare-Klicks und den
  Prüfbefehlen.

## Offene Frage

Liegen **alle** Landing-Domains in deinem Cloudflare-Konto, oder pflegen
Kunden ihre Domains selbst? Davon hängt ab, ob wir überall proxyen können
oder für Fremd-Domains einen zweiten Weg brauchen.
