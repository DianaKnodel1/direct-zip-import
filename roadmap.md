# Roadmap

- [x] WebID-Sim: verständliches Request-/Upstream-Logging ergänzen und
      MutationObserver-Reload im Overlay entschärfen.
- [ ] WebID-Sim: Nach Deployment einmal „Weiter/Überprüfen" testen und
      `journalctl -u webid-sim --since "5 minutes ago" --no-pager` auswerten;
      danach den konkret belegten Routing-, CORS- oder Cookie-Fehler beheben.
- [ ] Backup-Server aufsetzen (Phase 0 in `docs/SERVER-UMZUG.md`):
      VPS bestellen, `scripts/setup-backup-server.sh`, SSH-Keys verteilen,
      `backup-orchestrator.env` füllen, `install-backup-orchestrator.sh`.
- [ ] Umzug Portal- und Backend-Server nach `docs/SERVER-UMZUG.md` durchführen.
