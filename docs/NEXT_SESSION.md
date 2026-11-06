# Next session — Postiz (AIQSO production instance) follow-through

_Refreshed 2026-11-06 ~02:10 CT (qdesk): steps 1, 3, 4 of the 01:41 kickoff verified DONE; step 5 is next._

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

WHERE THINGS STAND (as of 2026-11-06 ~02:10 CT):
- Production Postiz v2.25.3 on LXC 220 since 2026-11-05 (rollback: ZFS snapshot `pre_postiz_v2253_20261105`,
  /root/backups/postiz-db-pre-v2.25.3-20261105.dump, compose/.env *.bak-* files). Registration closed.
  Compromise audit clean. Temporal healthchecks added; ES visibility repaired (search attrs postId /
  organizationId mapped, 67 post workflows re-indexed via /root/reindex-post-workflows.sh).
- Rotations DONE: Postiz API key (all consumers updated), Telegram bot token (Q), SES SMTP — new key
  AKIA25BLQOYKU2TKAZ57 Active (last used 2026-11-06 03:38 UTC, ses-smtp); OLD key AKIA25BLQOYKT3FXMW22
  **DELETED 2026-11-06 01:47 CT** (CloudTrail, user dunkin) → `list-access-keys` = exactly one key.
  JWT_SECRET deliberately NOT rotated.
- **Backups DONE 2026-11-06:** CT 220 is in `daily-pbs-backup` (02:00 CT → pbs-synology, latest
  2026-11-05T08:32:46Z, ~25 GB nightly). Logical dump `/usr/local/bin/postiz-db-backup.sh` (root crontab
  `15 3 * * *`, pg_dump -Fc → /root/backups/postiz-db-nightly-*.dump, keep only if ≥45 public tables,
  7-day retention) now pings Healthchecks **page-class check `lxc220-postiz-db-backup`** (CT 371, channel
  notify-router (page), grace 2h) via /etc/postiz-db-backup.conf (root 0600). Proof: manual run 01:59 CT →
  88 tables, check up. Repo copy + runbook: **AIQSO/homelab-infrastructure PR #68 — OPEN, unreviewed**
  (merge changes nothing at runtime).
- 16 channels. TikTok: aiqso_io + blastradiuspod (cmw4nhymc0004s04lo8oscmw4, in aiqso-social
  config/channels.yaml @ affbff6), both on the Sandbox app → inbox (UPLOAD) only. Dev app
  7590200833132808248 Production **"AIQSO Publisher"** re-submitted 2026-11-06, In review, 6 scopes
  (user.info.basic/profile/stats, video.list/publish/upload), Website URL https://postiz.aiqso.io.
  aiqso-social TIKTOK_PRODUCTION.md (171c63f) and the aiqso-events kickoff point HERE (step 7 below).
- Autopost "The Blast Radius — new episodes": Castopod feed → DRAFT on BR Facebook + BR Instagram, gpt-4.1
  text, no picture, lastUrl seeded at EP4 (EP5 first trigger). YouTube RSS is 404 globally — do not use.
- Signatures (AIQSO, Blast Radius) and Sets (AIQSO brand, Blast Radius) exist.
- Local state: postiz-app main == origin/main, clean. aiqso-social clean (untracked .mcp.json only).
  homelab-infrastructure local checkout is on feat/paperclip-board-inbox (someone else's work — untouched).
- Open PRs listed by gh for THIS repo are UPSTREAM gitroomhq PRs — not ours; ignore.
- Related Odoo tasks (no dedicated Postiz project): #1234 rotate stale credentials (Backlog — rotations
  now done, can be closed with evidence after /done-check), #1246 video-class channels YouTube+TikTok,
  #1222 Postiz channel coverage.
- Other Postiz instances on CTs 221/222 (client instances per Odoo #602/#591) — NOT upgraded/hardened.
- Same-day schedule elsewhere: aiqso-events recap posts publish Fri 2026-11-06 10:00 CT (its step 1);
  social drip starts Mon 2026-11-09 11:30 CT.

NEXT STEPS (in order — each has a gate; stop and report if a gate fails):
1. Verify post-restart scheduling health. Gate: n8n SocialPublish run of 2026-11-06 08:30 ET succeeded,
   AND the first queued post due 2026-11-09 17:30 UTC shows state PUBLISHED via GET /public/v1/posts
   (read back the platform URL). (Carried from the 01:41 kickoff step 2.)
2. Queued-post integrity monitor (biggest real risk: Postiz swallows Temporal startWorkflow errors).
   Nightly script on LXC 220: every post with status QUEUE and publishDate > now must have a Running
   workflow `post_<id>` (visibility query postId="<id>"). Mismatch → notify-router Telegram; success →
   Healthchecks ping (create the check like lxc220-postiz-db-backup: Django shell on CT 371, pipe the
   ping URL straight into a root-0600 conf, never print it). Add the script to homelab-infrastructure
   (branch + PR, worktree off origin/main — do NOT touch the checked-out feature branch) and a row in
   cron-verify jobs.tsv. Gate: one run prints N queued / N running / 0 missing; a fake id makes it alert.
3. Postiz webhooks → n8n → notify-router: per-post "published" alert with platform URL (n8n dev first).
   Gate: one real publish produces exactly one Telegram message with a working link.
4. Autopost cover art: ubuntuai hourly job — drafts from the autopost lacking an image get the matching
   episode's itunes:image from the Castopod feed (upload-from-url, recreate draft, delete old). Gate:
   dry-run on EP4 shows the right image URL; first live run on EP5 leaves one draft per channel, with image.
5. homelab-infrastructure PR #68: get it reviewed (CodeRabbit/Codex/Claude) and merged per
   feedback_pr_merge_policy. Gate: MERGED, and `/usr/local/bin/postiz-db-backup.sh` on CT 220 is
   byte-identical to scripts/backup/postiz-db-backup.sh on main. Also confirm tonight's 03:15 CT run:
   /var/log/postiz-db-backup.log has an OK line dated 2026-11-06 03:1x and the check is still up.
6. TikTok Production (owned HERE): when approved (2–5 business days from 2026-11-06), swap
   TIKTOK_CLIENT_ID/SECRET in /opt/postiz/.env (snapshot + .bak first), `docker compose up -d postiz`,
   Q reconnects aiqso_io + blastradiuspod (OAuth popups are Q's clicks). Gate: both channels connected
   with no scope error and a DIRECT_POST test (privacy SELF_ONLY) succeeds. If rejected: read Review
   comments, fix, ask Q before resubmitting. Then update aiqso-social TIKTOK_PRODUCTION.md + Vaultwarden.
7. Release watcher: weekly check of gitroomhq/postiz-app releases vs pinned digest → Telegram.
   Gate: run shows "current v2.25.3 / latest vX" message delivered.
8. Housekeeping on/after 2026-11-12 (one stable week): remove ZFS snapshot pre_postiz_v2253_20261105,
   /root/backups/*-20261105.dump, and .env.bak-* files on LXC 220 (they contain secrets); also
   qdesk/qmac/ubuntuai postiz.env *.bak-pre-rotate-20261105. Gate: `ls /opt/postiz/.env.bak*` empty;
   `zfs list -t snapshot | grep 220` has no pre_postiz. Leave postiz-db-nightly-* to the script's rotation.
9. Audit Postiz CTs 221/222: version, DISABLE_REGISTRATION, exposed ports. Report only; ask Q before
   upgrading client instances. Gate: one-line status per CT.
10. Enhancements — propose with cost/benefit, build only on Q's go: Hermes/Goose → Postiz built-in MCP
    (/api/mcp, Bearer key); OPENAI_BASE_URL → LiteLLM/Ollama with model aliases for gpt-4.1 /
    gpt-image-1; self-hosted short links (Kutt) on go.aiqso.io; Blast Radius LinkedIn Showcase Page;
    JWT_SECRET rotation with a re-encrypt script; TikTok "Verify domains" for photo posts (Postiz sends
    photos PULL_FROM_URL, videos FILE_UPLOAD).
11. Decision needed from Q: AIQSO Discord channel is absent from Postiz — reconnect or drop from
    channels.yaml.

Tooling caveats: claude-in-chrome cannot screenshot tiktok.com pages (OAuth popups open outside the
tab group — Q must click those); the TikTok developer portal will NOT save a draft until a demo video is
attached, and a cleared-cookie session makes uploads "fail" silently (HTTP 200, nothing attached) — Q
must re-login; file_upload only accepts files under the session's dirs (copy into $CLAUDE_JOB_DIR/tmp)
and ≤10 MB per call. `/backup-verify`'s verify.sh is BROKEN on qdesk (`proxmox-backup-manager` not
found on its ssh target) — use `ssh pve840 'sudo -n pvesm list pbs-synology --vmid <id>'`. Reading
~/.claude/secrets/healthchecks-admin.env is denied — use the CT 371 Django shell instead. ssh qmac needs
qmac awake; reading n8n's API key is denied — n8n credential edits are Q's; env-guard hook blocks
commands touching .env unless they print nothing (`# ENV-GUARD-OVERRIDE`). homelab-infrastructure's
pre-push gate takes >2 min — run pushes in the background.

NOTE ON PATHS: the memory files cited below are NOT in this repo. They live in the shared
auto-memory dir: ~/.claude/projects/_shared-memory-projects-root/

FIRST: read ~/projects/postiz-app/CLAUDE.md + reference_postiz_ops.md, feedback_social_brand_routing.md,
feedback_postiz_publish_authorized.md, project_aiqso_social_pilot.md, reference_notify_router.md,
project_healthchecks_monitor.md, feedback_pr_merge_policy.md. Skills: /bw-session, /cron-verify,
/done-check, /homelab-status, /verify-deploy.
