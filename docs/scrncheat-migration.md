# Multi-domain setup: Access policy + scrncheat.com migration

Operational runbook for the config change in [wrangler.jsonc](../wrangler.jsonc) that adds
`opentostage.com`, `pierrefouquet.co.uk`, and `scrncheat.com` (and the `mail.*` custom-domain
routes). These steps are **Cloudflare dashboard / DNS actions** — they are not captured in
`wrangler.jsonc` and must be done by hand.

Two facts drive everything below:

- Inbound delivery for `@scrncheat.com` is controlled by the **`MX` records on the apex
  `scrncheat.com`**, not on `mail.scrncheat.com`.
- `mail.scrncheat.com` is being **repurposed** as the app URL (the Worker custom domain). Its old
  use (Mail-in-a-Box host) must be removed first or it will conflict.

---

## 1. Cloudflare Access policy

**Goal:** all three hostnames — `mail.opentostage.com`, `mail.pierrefouquet.co.uk`,
`mail.scrncheat.com` — must be protected by **one** Access application sharing the **same
`POLICY_AUD`** the Worker validates. The Worker checks a single audience tag, so they cannot be
split across separate Access apps.

Deploy the new routes first (merge the config + `npm run deploy`, or add the custom domains under
the Worker) so the hostnames exist before attaching Access.

### Add the hostnames to the existing Access application
1. Go to [one.dash.cloudflare.com](https://one.dash.cloudflare.com) (Zero Trust).
2. **Access → Applications**.
3. Open the self-hosted application that currently protects `mail.pierrefouquet.co.uk` →
   **Configure**.
4. Under **Application Configuration → Application domain(s)**, add:
   - `mail.opentostage.com`
   - `mail.scrncheat.com`
5. Leave the **Policies** (who is allowed in) as-is — they now apply to all three hostnames.
6. **Save**.

### Confirm the audience tag is unchanged
- Adding domains to the same application does not change its **Application Audience (AUD) Tag**, so
  the Worker's `POLICY_AUD` secret stays valid. (Settings → **Application Audience (AUD) Tag** to
  confirm it matches.)
- `TEAM_DOMAIN` never changes — it is the team-wide Access domain.

### If you originally used one-click Access for Workers
One-click can create a separate app per route, producing multiple AUDs the Worker can't all
validate. If so, consolidate into a single self-hosted app covering all three hostnames, then set
the Worker secrets to that app's values:

```bash
wrangler secret put POLICY_AUD
wrangler secret put TEAM_DOMAIN
```

### Verify
Open each `https://mail.<domain>` in an incognito window → you should hit the Access login, then the
inbox. If any one loads **without** a login prompt, it is not covered yet.

---

## 2. Migrating scrncheat.com email off Mail-in-a-Box → this inbox

`scrncheat.com` email is currently hosted on **Mail-in-a-Box (MIAB)**, with `mail.scrncheat.com` as
the box host. We are cutting over to **Cloudflare Email Routing → the Worker** and retiring MIAB.

### Your deletion checklist: MIAB's "External DNS" page
MIAB admin → **System → External DNS** lists **every** record MIAB created for `scrncheat.com` —
the authoritative checklist. Remove these from Cloudflare (except where Email Routing replaces
them):

| MIAB record | Action on Cloudflare |
|---|---|
| **A/AAAA `mail.scrncheat.com`** → box IP | **Delete.** The Worker custom domain must own `mail.scrncheat.com`. |
| **MX `scrncheat.com`** → `mail.scrncheat.com` | Replaced by Email Routing's `route1/2/3.mx.cloudflare.net` (enable wizard offers the swap). |
| **SPF TXT** `v=spf1 mx -all` | Replace with `v=spf1 include:_spf.mx.cloudflare.net ~all` (strict `-all` → `~all`). |
| **DKIM TXT `mail._domainkey`** | **Delete** — MIAB's key dies with the box. Email Sending issues a new Cloudflare DKIM record. |
| **DMARC TXT `_dmarc`** | MIAB defaults to `p=quarantine`. Relax to `p=none` during cutover, re-tighten after. |
| **`autoconfig` / `autodiscover` A + SRV** (`_autodiscover._tcp`, `_imaps._tcp`, `_submission._tcp`, `_pop3s._tcp`) | **Delete** — they point clients at the dead box. |
| **CalDAV/CardDAV SRV** (`_caldavs._tcp`, `_carddavs._tcp`) | **Delete.** |

### ⚠️ The two that will silently break inbound mail if left stale
MIAB enables these to enforce TLS against the old box; if Cloudflare's mail servers can't satisfy
them, inbound mail **bounces**:

1. **MTA-STS** — delete `mta-sts.scrncheat.com` (A), the `_mta-sts.scrncheat.com` TXT policy, and
   the `_smtp._tls.scrncheat.com` (TLS-RPT) TXT.
2. **DANE / TLSA** — if DNSSEC was enabled, MIAB created `TLSA` records (e.g.
   `_25._tcp.mail.scrncheat.com`) pinning the box cert. **Delete** them, or DANE-validating senders
   refuse delivery. Also remove the **DS record** at your registrar if turning DNSSEC off (or
   re-sign via Cloudflare) — don't leave a DNSSEC chain pointing at MIAB's keys.

### Order of operations (avoid a delivery gap)
1. **Back up the mailbox first** over IMAP (Thunderbird or `imapsync`) while the box is alive —
   retiring is destructive.
2. Delete the old `mail.scrncheat.com` A/AAAA and the **MTA-STS + TLSA** records (the breakers).
3. Enable **Cloudflare Email Routing** on `scrncheat.com` → accept its MX swap → add a catch-all
   rule **→ Send to Worker** (the agentic-inbox Worker).
4. Fix **SPF**, delete old **DKIM**, relax **DMARC** to `p=none`.
5. Verify `scrncheat.com` for **Email Service** (outbound) → add the DKIM record Cloudflare gives.
6. Deploy so the Worker custom domain claims `mail.scrncheat.com`.
7. **Decommission the VPS/droplet** running MIAB once mail is confirmed flowing, and remove/fix the
   box's rDNS/PTR at the VPS host (that lives at the provider, not Cloudflare).

### Heads-up on reusing `mail.scrncheat.com`
That hostname serves MIAB's webmail (`/mail`) and admin (`/admin`) today. Once it points at the
Worker, those stop working — expected for a clean retirement. (To keep MIAB's webmail temporarily,
put the app on a different subdomain like `inbox.scrncheat.com` instead.)

### Verify the cutover
- `dig MX scrncheat.com` → Cloudflare only.
- `dig TLSA _25._tcp.mail.scrncheat.com` and `dig TXT _mta-sts.scrncheat.com` → **empty**.
- `dig +short mail.scrncheat.com` → resolves to the Worker custom domain.
- Test email **in** → lands in the inbox. Test **out** → delivered, passes SPF/DKIM.
