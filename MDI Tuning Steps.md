# MDI Alert Tuning — Customer Meeting Playbook

A step-by-step agenda and working guide for running an MDI alert tuning session with a customer. Designed for ~60–90 minutes with a follow-up cadence.

---

## Pre-Meeting Prep (do before the call)

1. **Pull health snapshot** (Defender XDR → Settings → Identities → Health issues): export Global + Sensor tabs.
2. **Export current alert volume** — last 30 days, grouped by detection name and by entity type. Flag any alert where the actor resolves as an **IP instead of a host/user** (NNR red flag).
3. **Pull current tuning state**:
   - Existing Alert Tuning rules (Defender XDR → Settings → Microsoft Defender XDR → Rules → Alert tuning)
   - Legacy MDI exclusions (Settings → Identities → Excluded entities, both tabs)
   - Alert threshold page (Settings → Identities → Adjust alert thresholds) — note any non-**High** settings
4. **Pull posture/Secure Score** items for Identity.
5. **Confirm sensor coverage** — all DCs (including RODCs, AD FS, AD CS). Note any gaps.

Bring all of the above to the meeting as a one-page "current state" view.

---

## Meeting Agenda

### Step 1 — Frame the session (5 min)

State the goal clearly: *"Reduce noise without losing fidelity, using Microsoft's recommended order: fix root cause → tag entities → tune at the detector level → exclude/threshold only as a last resort."*

Set the ground rule: **no tuning on alerts whose root cause is a health or NNR issue.**

---

### Step 2 — Validate readiness (10 min)

Walk the customer through the baseline checklist before touching any rules:

- [ ] Sensors on **every DC** (including RODCs); AD FS / AD CS sensors where applicable
- [ ] All sensors **Healthy** in Defender XDR
- [ ] **NNR functional** — spot-check a few recent alerts to confirm actors resolve to hostnames/users, not IPs
- [ ] Correct **capture NIC** selected on each sensor
- [ ] **Learning periods complete** for detections you plan to tune (see table below)
- [ ] Secure Score Identity recommendations reviewed

If anything in this list fails, stop and fix it first. Tuning before this is complete bakes bad behavior into the rules.

**Key learning periods to check:**

| Detection | Learning period |
|---|---|
| Network-mapping recon (DNS) | 8 days |
| User/group recon (SAMR) | 4 weeks per DC |
| Golden Ticket (enc downgrade) | 5 days |
| Additions to sensitive groups | 4 weeks per DC |
| Brute force (Kerberos/NTLM) | 1 week |
| LDAP principal recon | 15 days per machine |
| Over-PtH (forced enc) | 1 month |
| Suspicious VPN | 30 days, ≥5 connections |

---

### Step 3 — Classify the current alert backlog (15 min)

Work through the top 10–20 noisiest alerts with the customer. For each, assign:

- **TP** — real, keep/escalate
- **B-TP** — real but expected (scanner, admin automation, legacy app)
- **FP** — didn't happen / detection logic misfire

This classification **drives every tuning decision** that follows. Don't skip it.

Microsoft note to share: *a spike of identical alerts usually indicates a configuration issue, not a detection problem — investigate before suppressing.*

---

### Step 4 — Apply the tuning decision tree (20–30 min)

For each noisy alert, walk this order with the customer. Stop at the first answer that fits.

1. **Is it caused by a health / NNR / sensor issue?** → Fix the root cause. No tuning.
2. **Is the entity a known sensitive or honeypot asset?** → Apply an **entity tag** (Sensitive, Honeytoken, Exchange Server) instead of suppressing.
3. **Can the B-TP be described by evidence (user, device, IP, process)?** → Create a **Defender XDR Alert Tuning rule** scoped to that evidence. This is the preferred path.
4. **Does a scanner/service need silencing for one specific detection only?** → **Per-rule exclusion** under Settings → Identities → Excluded entities.
5. **Does an entity need silencing across every detection?** → **Global excluded entity** (use sparingly).
6. **Is the alert threshold itself wrong for the environment (e.g., NAT/VPN causing Identity Theft noise)?** → Adjust the threshold, but **only after careful review**. Never use *Recommended Test Mode* in production.
7. **Is the problem really "too many people seeing this alert" rather than "too many alerts"?** → Use **Unified RBAC / scoped access** to tier visibility by domain, data source, or SOC role — don't delete alerts the SOC still needs.

**Guardrails to state explicitly:**

- Never broadly suppress DCSync, credential theft, Suspected identity theft, lateral movement path, or replication abuse detections. Scope them narrowly if needed, don't silence them.
- Never suppress by category (e.g., all "Reconnaissance"). Tune at the detector level only.
- Alert tuning rules beat exclusions because they're more granular and leave an audit trail of what was suppressed.

---

### Step 5 — Build the rules live (15 min)

Pick 2–3 high-impact B-TPs from Step 3 and build the tuning rules together in the portal. Demonstrating once ensures the customer can repeat it.

For each rule built:

- Capture the **rule name, detector, scope/evidence, justification, owner, review date** in a tracker (spreadsheet or ticket).
- Confirm the detector appears under Alert tuning after saving.
- If possible, run a **controlled validation** (e.g., rerun the scanner, have the service account perform the expected action) and confirm the alert is auto-resolved by the rule — not missing entirely.

---

### Step 6 — Replace legacy exclusions (10 min)

For each existing legacy exclusion the customer has:

1. Identify the MDI detection it targets.
2. Determine whether it's still needed (or if the source behavior has changed).
3. Rebuild it as an **Alert Tuning rule** if possible.
4. Remove the legacy exclusion once the tuning rule is validated.
5. Document the migration.

Default MDI-shipped exclusions for *Suspicious communication over DNS (2031)* can stay unless the customer has a reason to remove them.

---

### Step 7 — Establish cadence and ownership (10 min)

Agree on who owns tuning and when it's reviewed. Microsoft's recommended cadence:

- **Daily** — triage high-risk users and new alerts; create tuning rules for confirmed B-TP/FP patterns as they emerge.
- **Weekly** — review alert patterns and investigation priority score trends.
- **Monthly** — review every tuning rule: does it still match? zero-match rules should be deleted. Review posture assessments and sensor/version updates. Track Message Center and *What's new in Defender for Identity* for new detections that may need scoping.

Capture owners for each cadence. Put the monthly review on the calendar before ending the meeting.

---

### Step 8 — Known product constraints to set expectations (5 min)

Call these out so the customer isn't surprised later:

- Alert tuning conditions rely on **entity type**; if NNR is unhealthy, actors surface as IPs and rules won't match. Fix NNR first.
- Some MDI response actions don't surface uniformly across all XDR automation/hunting tables — plan hunting queries accordingly.
- Alerts appear in both **classic** and **Defender XDR** formats during the transition; tuning rules apply to both, alert IDs stay stable across formats.
- **Recommended Test Mode** is evaluation-only, time-boxed (max 60 days), and not for production.
- Don't "tune around" a product gap — raise it through support so it's fixed at the source.

---

### Step 9 — Close with a written action list (5 min)

Before ending, capture:

1. Health/NNR issues to fix (owners + dates)
2. Entity tags to apply (users, devices, honeytokens)
3. Tuning rules built today + tuning rules to build this week
4. Legacy exclusions to migrate
5. Threshold changes considered (and justification if any)
6. Next review date
7. Open questions to raise with Microsoft support / product group

---

## Reference Links

- [Tune an alert (Defender XDR)](https://learn.microsoft.com/defender-xdr/investigate-alerts#tune-an-alert)
- [MDI detection exclusions](https://learn.microsoft.com/defender-for-identity/exclusions)
- [Adjust alert thresholds](https://learn.microsoft.com/defender-for-identity/advanced-settings)
- [Daily ops guide](https://learn.microsoft.com/defender-for-identity/ops-guide/ops-guide-daily) / [Weekly ops guide](https://learn.microsoft.com/defender-for-identity/ops-guide/ops-guide-weekly) / [Monthly ops guide](https://learn.microsoft.com/defender-for-identity/ops-guide/ops-guide-monthly)
- [Security testing best practices (learning periods)](https://learn.microsoft.com/defender-for-identity/security-testing-best-practices)
- [Understanding and managing alerts](https://learn.microsoft.com/defender-for-identity/understanding-security-alerts)
