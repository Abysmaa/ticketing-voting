# Ticketing SaaS — Admin Panel Spec v1

Deep dive on the Admin/platform-owner module referenced in `PRODUCT_SPEC.md` sect. 4 and `TRD.md` sect. 4/5. Page-by-page requirement breakdown — no wireframe here, doc only. Feeds the wireframe pass once this is confirmed.

## 1. Scope & Access Model
- Admin is a **separate account pool** from Organizer/Attendee — `admin` entity, no public signup (TRD sect. 4). First admin account created by direct DB seed / internal provisioning script.
- v1 has **one flat admin role** — no granular permission tiers (e.g. no "read-only admin" vs "super admin" split). Anyone with an admin account can do everything on this page list. Revisit if the team grows past a handful of platform staff.
- Session should time out shorter than organizer/attendee (financial + PII data exposure) — recommend 30 min idle, confirm with team.
- **2FA strongly recommended** for admin login given access to payout bank info and KYC documents — not building it is a real risk, flag as open decision (build vs skip for v1).
- **Every mutating admin action must write to an audit log** (sect. 6) — non-negotiable on a platform that moves other people's money.

## 2. Page List (summary)

| # | Page | Purpose | v1 Priority |
|---|---|---|---|
| 1 | Admin Login | Internal-only auth entry | Must — built (shared Flow 0) |
| 2 | Platform Overview | Top-line KPI + sales trend | Must — built |
| 3 | Organizer List | Search/filter all organizer | Must — built |
| 4 | Organizer Detail | Drill into one organizer, act on it | Must — built |
| 5 | KYC Review Queue | Approve/reject pending verification | Must — built |
| 6 | Commission & Payout Queue | See/settle what's owed each organizer | Must **if payout is manual** (TRD sect. 7 open decision) — built |
| 7 | Audit Log | Trail of every admin action | Must — built |
| 8 | Events Oversight | All events platform-wide, emergency unpublish | Should — not built |
| 9 | Orders / Transactions Oversight | All orders platform-wide, dispute lookup | Should — not built |
| 10 | Disputes Log | Manual tracking of flagged issue | Should (cheap, low-effort) — not built |
| 11 | Admin Settings | Own profile, platform config | Could — not built |

"Must" = blocks the admin module from being minimally useful for its stated PRD purpose ("oversee organizer, take commission, handle dispute"). "Should"/"Could" can trail v1 launch without blocking it.

## 3. Page-by-Page Detail

### 3.1 Admin Login
- **Content**: email + password fields, optional 2FA code field (if built). No "Sign up" link — contrast with Organizer/Attendee Login (Flow 0), reinforces no self-registration.
- **Requirement**: every login attempt (success + failure) logged. Rate-limited harder than organizer/attendee login (smaller, higher-value target).

### 3.2 Platform Overview
- **Content**:
  - KPI cards: Total GMV (toggle all-time / this month), Platform Commission Earned, Active Organizer count, Verified-vs-Pending organizer split, Live Event count, Total ticket sold.
  - Sales-over-time chart, platform-wide, date-range filterable.
  - "Top Organizers by Revenue" mini table (top 5–10).
  - Recent activity feed: new organizer signup, new event published, recent refund — quick pulse, not a full log.
  - Quick-link cards to Organizer List, KYC Queue (w/ pending-count badge), Disputes Log (w/ open-count badge).
- **Requirement**: aggregation queries need `order.created_at` + `organizer_id` indexed; date-range filter; IDR currency formatting at scale (billions).

### 3.3 Organizer List
- **Content** (table columns): name, email, plan tier, verification status, event count, lifetime revenue, signup date, active/suspended status.
- **Filter/search**: by name/email, plan tier, verification status, active/suspended.
- **Actions**: View → Organizer Detail; Suspend/Reactivate (confirm modal + required reason note); Export CSV.
- **Requirement**: pagination (organizer count can grow into the thousands), search must stay fast at scale.

### 3.4 Organizer Detail (new — not built as a concept until now)
- **Content**:
  - Profile: name, email, signup date, contact.
  - KYC panel: tax_id, uploaded document (view/download), verification status + **Approve/Reject action** (reject requires a reason, notifies organizer to resubmit).
  - Payout bank account — **masked** display (e.g. "••• 4821"), same convention as the organizer's own Settings page.
  - Plan tier + **commission_rate override** (TRD sect. 4 already has this field on `organizer` — this is the page that edits it).
  - Their events: title, status, date, ticket sold, revenue — read-only list, links out to that event's own Sales Summary.
  - Suspend/Reactivate action, with a mini audit trail scoped to this organizer (pulled from the global Audit Log, sect. 3.7).
- **Requirement**: sensitive fields (bank account, tax ID) need access control on top of admin-auth — consider whether viewing them (not just changing them) should also hit the audit log. Flag as open question, not decided here.

### 3.5 KYC Review Queue
- **Content**: table of **pending-only** verification submissions — organizer name, tax_id, submitted document (thumbnail/preview), submitted date. Sorted oldest-first.
- **Actions**: Approve / Reject (reject requires reason, triggers organizer notification).
- **Requirement**: needs a document viewer (PDF/image inline preview, not just a download link). **Open question**: target review turnaround time (SLA) — affects whether this needs alerting/escalation later.

### 3.6 Commission & Payout Queue
- **Content** — depends on TRD sect. 7's still-open "manual vs automatic disbursement" decision:
  - **If manual** (likely simplest for v1): per-organizer pending payout balance (revenue earned − commission − already paid out), list of organizers awaiting payout with bank details, **Mark-as-Paid** action (reference number + date, creates a payout record), per-organizer payout history.
  - **If automatic** (gateway split-payment): this page downgrades to a **read-only reconciliation view** — gateway handles the actual transfer, admin just confirms numbers match.
- **Requirement**: financial-critical — every payout action logged to Audit Log with reference number. Gated on `organizer.verification_status = verified` (already decided, TRD sect. 4/ROADMAP Phase 5).
- **This page is likely the single highest-value "Must" item after Organizer List** — without it, an organizer on manual payout has literally no way to get paid.

### 3.7 Audit Log
- **Content**: chronological log — admin (who), action (what: "suspended organizer X", "approved KYC for Y", "force-refunded order Z", "marked payout paid for W"), timestamp, optional note/reason.
- **Requirement**: every mutating admin action across every other page in this doc writes here. New entity needed: `admin_audit_log` (admin_id, action_type, target_type, target_id, note, created_at). Filterable by admin, action type, date range, target.

### 3.8 Events Oversight
- **Content**: table of **all events platform-wide** — title, organizer, category, type (single/multi-day), status, date, ticket sold, revenue.
- **Filter**: status, category, organizer, date range.
- **Actions**: View (read-only, links to organizer's own event dashboard); **emergency Unpublish** (policy violation / fraud response).
- **Distinction from the already-deferred "event moderation"**: the deferred item (TRD sect. 8) is **pre-publish review** — a queue an event sits in before it can go live. This page is **post-publish enforcement** — pulling down something already live. Recommend keeping the Unpublish action even though full pre-publish moderation stays deferred; it's a cheap safety net, not the same scope as the deferred item.

### 3.9 Orders / Transactions Oversight
- **Content**: all orders platform-wide — order id, buyer, event, organizer, amount, status, gateway ref, date.
- **Filter/search**: order id, buyer email, event, organizer, status, date range.
- **Actions**: View detail; **force-refund** (same underlying action as the organizer's own refund, sect. 5 endpoint, but admin-triggered — needed for dispute resolution even without a full case-management workflow).
- **Requirement**: this is the page that actually lets admin "handle disputes" in a lightweight way (search the order, look at it, force a refund if warranted) — even with the Disputes Log (3.10) staying a manual tracker rather than a workflow engine.

### 3.10 Disputes Log
- **Content**: manually-maintained internal list — order id, organizer, complaint description, status (open/resolved), notes. No automated submission form from organizer/attendee side in v1 — admin logs it after handling a dispute through whatever external channel (email, WhatsApp, phone) it came in on.
- **Requirement**: minimal — basically a lightweight internal CRM entry, not a support ticketing system. Cheap to build, worth including even though the full "dispute-resolution workflow" (TRD sect. 8) stays deferred.

### 3.11 Admin Settings
- **Content**:
  - Own profile: name, email, password change.
  - Manage other admin accounts — **open decision**: v1 keeps this DB/seed-script-only (simplest, matches "no public signup" cleanly) vs building an in-product "invite admin" flow. Recommend DB-only for v1, revisit if platform team grows.
  - Platform config: default commission rate, free-tier limits (event/attendee cap — these are the exact numbers still marked TBD in PRD sect. 7 / TRD sect. 7), category taxonomy list. Making these admin-editable removes a redeploy cycle every time the business changes a number — valuable, but can also just be a config file for v1 if that's simpler to ship. Flag as a build-vs-hardcode tradeoff, not decided here.

## 4. New/Updated Data Model (beyond what TRD.md sect. 4 already has)

| Entity | Fields | Note |
|---|---|---|
| `admin_audit_log` | id, admin_id (FK), action_type, target_type, target_id, note, created_at | Every mutating admin action writes one row here |
| `payout_record` | id, organizer_id (FK), amount, reference_number, paid_at, marked_by_admin_id (FK) | Only needed if manual disbursement (sect. 3.6) is the chosen path |
| `dispute_log` | id, order_id (FK, nullable), organizer_id (FK, nullable), description, status, created_by_admin_id (FK), created_at | Backs sect. 3.10 |
| `organizer.commission_rate` | *(already exists, TRD sect. 4)* | This doc's Organizer Detail page (3.4) is the UI that edits it |

## 5. Cross-Cutting Requirements
- No public admin signup — DB seed / internal provisioning only.
- 2FA recommended, not yet decided (flag for team).
- Every mutating action → `admin_audit_log`.
- Shorter session timeout than organizer/attendee.
- Sensitive data (bank account, tax ID, KYC doc) masked/gated beyond plain admin-auth — exact policy still open (sect. 3.4).
- Rate limiting on admin login, harder than organizer/attendee (smaller, higher-value target — same principle as TRD sect. 6 NFR, applied to this pool too).

## 6. Recommended Phasing (maps to ROADMAP.md Phase 6a)
- **Phase 6a (v1, "Must")**: Admin Login, Platform Overview, Organizer List, Organizer Detail, KYC Review Queue, Audit Log, Commission & Payout Queue (if manual disbursement is chosen).
- **Phase 6b (post-launch, "Should/Could")**: Events Oversight, Orders/Transactions Oversight, Disputes Log, Admin Settings (platform config editing — can hardcode initially).

This is a bigger Phase 6a than the two-screen version currently wireframed (Platform Overview + Organizer List only). Recommend re-scoping Phase 6a in `ROADMAP.md` once this doc is confirmed, before moving to wireframe.
