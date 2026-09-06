# Notfall-Architektur: Backup-Server als einzige Reserve

## Entscheidung

Kein dauerhaft laufender Ersatzserver. Stattdessen:

- **Nur ein kleiner Backup-Server** (2 vCPU, 2–4 GB RAM, ~45 GB SSD, günstigster KVM-Tarif)
- Backup-Server bei einem **anderen Anbieter** als die Produktionsserver (sonst schützt er nicht gegen Sperrung/Totalausfall des Anbieters)
- Im Notfall: neue Server bestellen und aus dem Backup wiederherstellen

```text
Normalbetrieb                        Notfall (Server down/gesperrt)
-----------                          -------------------------------
Portal .124    ──┐                    1. Neue Server bestellen (bei beliebigem
Backend .123   ──┼── ziehen alle        anderem Anbieter)
Landing .234   ──┘   6 h Backups     2. Restore aus Backup-Archiv
                     │               3. DNS in Cloudflare auf neue IPs
                     ▼               4. Verifizieren
              Backup-Server                          ▲
              (anderer Anbieter) ────────────────────┘
```

## Warum das reicht

- Datenbank ist nur ~350 MB — Backup-Archiv klein, Restore schnell
- Sämtliche Skripte existieren bereits: `scripts/backup-orchestrator.sh`,
  `scripts/install-backup-orchestrator.sh`, `scripts/restore.sh`
- Wiederanlaufzeit im Notfall: ca. 4–8 h (statt 1–2 h mit Leerstand-Ersatzserver,
  der sonst nur Kosten verursacht)
- Maximaler Datenverlust: 6 Stunden (Backup-Intervall)

## Schritte (auf dem neuen Backup-Server)

1. Kleinsten KVM-Tarif bestellen (Ubuntu 22.04/24.04, 2 vCPU, 2–4 GB RAM,
   40–50 GB SSD) — anderer Anbieter als Portal/Backend/Landing.
2. `bash scripts/setup-backup-server.sh` (legt Backup-Nutzer und
   `/var/backups/portal/{daily,monthly,logs}` an).
3. Auf Portal, Backend, Landing, Bot und WebID jeweils den SSH-Key des
   Backup-Servers in `/root/.ssh/authorized_keys` eintragen.
4. `cp scripts/backup-orchestrator.env.example scripts/backup-orchestrator.env`
   und eintragen: `BACKUP_DIR`, `DB_HOST` (Backend .123), `DB_CONTAINER=supabase-db`,
   `SERVER_PORTAL_HOST`, `SERVER_LANDING_HOST`, `SERVER_BOT_HOST`,
   `SERVER_WEBID_HOST`, `BACKUP_RETENTION_DAYS`.
5. `bash scripts/install-backup-orchestrator.sh` — sichert automatisch alle 6 h.
6. Erstlauf: `bash scripts/backup-orchestrator.sh full`, danach
   `ls -lt /var/backups/portal/daily/ | head` prüfen. Status erscheint im Portal
   unter `/admin/infrastructure`.
7. Empfohlen: Verschlüsselung mit `age` aktivieren und privaten Schlüssel in den
   Passwortmanager legen.

## Notfall-Ablauf (dann, wenn es passiert)

Vollständig dokumentiert in `docs/DISASTER-RECOVERY.md`:

1. Neue Server bestellen (Backend 4 vCPU/8 GB, Portal/Landing je 2 vCPU/4 GB —
   oder im Notfall alle drei auf einem Server mit 3 vCPU/6 GB/80 GB + Swap).
2. Neuestes Archiv vom Backup-Server holen.
3. `bash scripts/restore.sh <archiv>.tar.gz` — spielt DB, Storage, Configs zurück.
4. DNS-A-Records in Cloudflare auf die neuen IPs (TTL vorher 300 setzen).
5. Prüfen: Login, Landing-Page, Testbewerbung, Mailversand.

## Keine Code-Änderungen nötig

Alle Skripte und Doku sind im Repo vorhanden. Es gibt nichts am Code zu bauen —
nur die Einrichtung auf dem neuen Server nach obigen Schritten.
