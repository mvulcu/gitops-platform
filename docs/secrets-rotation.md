# Secrets & Rotation Runbook

## Telegram Alertmanager (prod)

- Canonical store: Bitwarden → `prod / lingualink / telegram / alertmanager / bot-token`
- K8s: SealedSecret `apps/infra/monitoring/alertmanager-telegram-sealed.yaml`
- Secret name: `alertmanager-telegram` (ns: `monitoring`)

### Rotate procedure

1. In BotFather:
   - `/revoke <old token>`
   - `/token` → get a new token.
2. In Bitwarden:
   - Update the `prod / lingualink / telegram / alertmanager / bot-token` entry.
3. On VPS:
   - Update `/tmp/alertmanager-telegram-secret.yaml` (stringData.bot-token).
   - `kubeseal ... > /tmp/alertmanager-telegram-sealed.yaml`.
4. In gitops-platform:
   - Update `apps/infra/monitoring/alertmanager-telegram-sealed.yaml`.
   - `git commit && git push`.
5. On VPS:
   - `flux reconcile ...`
6. Verification:
   - Generate a test alert → make sure the notification was received.