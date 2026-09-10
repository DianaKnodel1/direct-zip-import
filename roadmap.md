# Roadmap

- [x] WebID-Sim: verständliches Request-/Upstream-Logging ergänzen und
      MutationObserver-Reload im Overlay entschärfen.
- [ ] WebID-Sim: Nach Deployment einmal „Weiter/Überprüfen" testen und
      `journalctl -u webid-sim --since "5 minutes ago" --no-pager` auswerten;
      danach den konkret belegten Routing-, CORS- oder Cookie-Fehler beheben.
- [~] Backup-Server aufsetzen (läuft, Schritt 1–3 von 7): Skripte aufs Repo, SSH-Key
      verteilen, `backup-orchestrator.env` füllen, `install-backup-orchestrator.sh`.
      IPs trägt der User selbst ein (bleiben geheim).
- [ ] Umzug Portal- und Backend-Server nach `docs/SERVER-UMZUG.md` durchführen.
