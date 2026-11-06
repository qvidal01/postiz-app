# Next session — Postiz (AIQSO production instance) follow-through

ROLE: Senior platform/SRE engineer continuing operations of AIQSO's self-hosted Postiz social
scheduler (local fork repo: ~/projects/postiz-app — reference only; production runs the UPSTREAM
image, not this fork). Authorized to operate autonomously (ssh, compose changes with backups,
commit/push to infra/config repos, n8n dev workflows) with the safety rails below. Work evidence-first.

CONTEXT: Production Postiz v2.25.3 on LXC 220 (pve840, rootfs on ZFS nvme-fast → `pct snapshot`
works), /opt/postiz, docker compose with image pinned by digest
(sha256:1e376ff2…ddda13), pm2 inside the `postiz` container (backend / frontend / orchestrator),
Temporal 1.29.4 + Postgres + Elasticsearch 7.17 visibility (ES data now on a persistent volume).
Public: https://postiz.aiqso.io, public API /api/public/v1 (needs browser User-Agent + raw key in
Authorization). There is NO CD — every change is a manual ssh change on LXC 220.
Safety rails:
- `pct snapshot 220 <name>` before any compose/image/DB change; back up .env/compose as *.bak-<what>-<date>.
- Never print .env values or keys. Read secrets from Vaultwarden via bw-session
  (item `PostIZ` field `API`; item `AWS SES SMTP - Postiz`), pipe them over stdin.
- Never set DISABLE_SSRF_PROTECTION. DISABLE_REGISTRATION=true stays.
- Restart the orchestrator ONLY via /usr/local/bin/postiz-orchestrator-restart.sh (plain
  `pm2 restart` orphans the old process → EADDRINUSE :3002 crash loop). .env changes need
  `docker compose up -d postiz` (env_file).
- Brand routing: podcast → Blast Radius accounts only; AIQSO → AIQSO only; personal accounts manual-only.
- Publishing real posts: Q validates content + media first (feedback_postiz_publish_authorized).

WHERE THINGS STAND (as of 2026-11-06):
- Upgraded to v2.25.3 on 2026-11-05 (rollback: ZFS snapshot `pre_postiz_v2253_20261105`, DB dump
  /root/backups/postiz-db-pre-v2.25.3-20261105.dump, compose/.env *.bak-* files). Registration closed.
  Compromise audit clean. Temporal healthchecks added; ES visibility repaired (custom search attrs
  postId/organizationId mapped, 67 post workflows re-indexed via /root/reindex-post-workflows.sh).
- Rotations DONE 2026-11-05/06: Postiz API key (all consumers updated: ubuntuai
  ~/scripts/.social-draft.env, qdesk + qmac ~/.claude/secrets/postiz.env, qmac postiz-mcp.sh, n8n
  credential by Q); Telegram bot token (Q); SES SMTP — new IAM key for aiqso-postiz-ses active,
  OLD key AKIA25BLQOYKT3FXMW22 DELETED 2026-11-06 01:47 CT. JWT_SECRET deliberately NOT rotated.
- 16 channels. TikTok: aiqso_io + blastradiuspod (integration cmw4nhymc0004s04lo8oscmw4), both
  Sandbox → inbox (content_posting_method UPLOAD) only. TikTok dev app 7590200833132808248
  Production "AIQSO Publisher" re-submitted for review 2026-11-06 with all 6 scopes Postiz requests
  (user.info.basic/profile/stats, video.list/publish/upload), Website URL https://postiz.aiqso.io.
- Autopost "The Blast Radius — new episodes": Castopod feed
  https://podcast.theblastradius.io/@blastradius/feed.xml → DRAFT on BR Facebook + BR Instagram,
  gpt-4.1 text, no picture, lastUrl seeded at EP4 (EP5 is the first that triggers). YouTube RSS is
  404 globally — do not use it.
- Signatures (AIQSO, Blast Radius) and Sets (AIQSO brand, Blast Radius) exist.
- aiqso-social channels.yaml BR tiktok line committed (affbff6); TIKTOK_PRODUCTION.md now points here (step 8). This repo (postiz-app) is clean.
- Open PRs listed by gh for this repo are UPSTREAM gitroomhq PRs — not ours; ignore.
- Related Odoo tasks (no dedicated Postiz project): #1234 rotate stale credentials (Backlog),
  #1246 video-class channels YouTube+TikTok, #1222 Postiz channel coverage.
- Other Postiz instances exist on CTs 221/222 (client/side instances per Odoo #602/#591) — they
  were NOT upgraded or hardened in this round.

NEXT STEPS (in order — each has a gate; stop and report if a gate fails):
1. ~~Commit + push aiqso-social config/channels.yaml (BR tiktok line)~~ **DONE** — affbff6 on origin/main (2026-11-06 01:46 CT), `config/` clean (verified 2026-11-06 from the aiqso-events session). Original step:
   Gate: `git -C ~/projects/aiqso-social status --porcelain config/` empty and the line is on origin/main.
2. Verify post-restart scheduling health. Gate: n8n SocialPublish run of 2026-11-06 08:30 ET
   succeeded, AND the first queued post due 2026-11-09 17:30 UTC shows state PUBLISHED via
   GET /public/v1/posts (read back the platform URL).
3. ~~Delete the inactive SES key.~~ **DONE** — CloudTrail: DeleteAccessKey on AKIA25BLQOYKT3FXMW22 by `dunkin` 2026-11-06 01:47 CT (right after the 01:41 handoff). Gate verified 2026-11-06 07:55 UTC: new key AKIA25BLQOYKU2TKAZ57 last used 2026-11-06 03:38 UTC, ServiceName ses-smtp (before the delete); `list-access-keys` = exactly one Active key. Original step: Gate first: `aws iam get-access-key-last-used` on the NEW key shows
   ServiceName ses-smtp with a date after 2026-11-06 01:00 UTC; then delete AKIA25BLQOYKT3FXMW22;
   gate: `aws iam list-access-keys --user-name aiqso-postiz-ses` lists exactly one Active key.
4. ~~Backups.~~ **DONE 2026-11-06 02:00 CT.** PBS: CT 220 in daily-pbs-backup 02:00 → pbs-synology, latest 2026-11-05T08:32:46Z (~25 GB, nightly since 11-02). Dump: /usr/local/bin/postiz-db-backup.sh (root crontab 15 3 * * *, 7-day retention, >=45-table gate) now pings Healthchecks page-class check `lxc220-postiz-db-backup` via /etc/postiz-db-backup.conf (root 0600); manual run 01:59 CT → 88 tables, check up (2 pings). Repo copy + runbook: AIQSO/homelab-infrastructure PR #68 (open). Caveat: /backup-verify script is broken here (`proxmox-backup-manager` not found on its ssh target) — checked via `pvesm list pbs-synology --vmid 220` on pve840 instead. Original step: Gate: /backup-verify shows a successful PBS backup of CT 220 within 24h; add a nightly
   `pg_dump -h localhost` of postiz-db to /root/backups with 7-day retention + a healthchecks ping;
   gate: dump file from tonight exists and `pg_restore -l` lists ≥45 tables.
5. Queued-post integrity monitor (biggest real risk: Postiz swallows Temporal startWorkflow errors).
   Nightly script on LXC 220: every post with status QUEUE and publishDate > now must have a Running
   workflow `post_<id>` found via visibility query postId="<id>". Mismatch → notify-router Telegram;
   success → healthchecks ping. Register in cron-verify jobs.tsv. Gate: run it once, prints
   N queued / N running / 0 missing; then deliberately query a fake id and confirm it alerts.
6. Postiz webhooks → n8n → notify-router: per-post "published" alert with platform URL (n8n dev
   first). Gate: one real publish produces exactly one Telegram message with a working link.
7. Autopost cover art: ubuntuai hourly job — find drafts created by the autopost lacking an image,
   take the matching episode's itunes:image from the Castopod feed, recreate the draft with the image
   via public API (upload-from-url, then delete the old draft). Gate: dry-run against EP4 data shows
   the right image URL; first live run on EP5 leaves exactly one draft per channel, with image.
8. TikTok Production: when approved, swap TIKTOK_CLIENT_ID/SECRET in /opt/postiz/.env (snapshot +
   .bak first), `docker compose up -d postiz`, Q reconnects aiqso_io + blastradiuspod. Gate: both
   channels connected with no scope error and a DIRECT_POST test (privacy SELF_ONLY) succeeds.
   If rejected: read Review comments, fix, ask Q before resubmitting.
9. Release watcher: weekly check of gitroomhq/postiz-app releases vs pinned digest → Telegram.
   Gate: run shows "current v2.25.3 / latest vX" message delivered.
10. Housekeeping on/after 2026-11-12 (one stable week): remove ZFS snapshot
    pre_postiz_v2253_20261105, /root/backups/*-20261105.dump, and .env.bak-* files on LXC 220
    (they contain secrets); also qdesk/qmac/ubuntuai postiz.env *.bak-pre-rotate-20261105.
    Gate: `ls /opt/postiz/.env.bak*` empty; `zfs list -t snapshot | grep 220` has no pre_postiz.
11. Audit Postiz CTs 221/222: version, DISABLE_REGISTRATION, exposed ports. Report only; ask Q before
    upgrading client instances. Gate: one-line status per CT.
12. Enhancements — propose with cost/benefit, build only on Q's go: Hermes/Goose → Postiz built-in MCP
    (/api/mcp, Bearer key); OPENAI_BASE_URL → LiteLLM/Ollama with model aliases for gpt-4.1 /
    gpt-image-1; self-hosted short links (Kutt) on a go.aiqso.io domain; Blast Radius LinkedIn
    Showcase Page (check Postiz linkedin-page can see it); JWT_SECRET rotation with a re-encrypt script.
13. Decision needed from Q: AIQSO Discord channel is absent from Postiz — reconnect or drop from
    channels.yaml.

Tooling caveats: claude-in-chrome cannot screenshot tiktok.com pages (OAuth popups open outside the
tab group — Q must click those); ssh qmac needs qmac awake (it timed out once); reading n8n's API
key is denied by permission rules — n8n credential edits are Q's; env-guard hook blocks commands
touching .env unless they print nothing (`# ENV-GUARD-OVERRIDE`).

NOTE ON PATHS: the memory files cited below are NOT in this repo. They live in the shared
auto-memory dir: ~/.claude/projects/_shared-memory-projects-root/

FIRST: read ~/projects/postiz-app/CLAUDE.md + reference_postiz_ops.md, feedback_social_brand_routing.md,
feedback_postiz_publish_authorized.md, project_aiqso_social_pilot.md, reference_notify_router.md,
project_healthchecks_monitor.md. Skills: /bw-session, /backup-verify, /cron-verify, /done-check,
/homelab-status.
