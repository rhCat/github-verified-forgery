---
layout: default
title: "I committed as Linus Torvalds with a green Verified badge on GitHub"
---

# I committed as Linus Torvalds with a green "Verified" badge on GitHub

GitHub's own documentation establishes a chain of trust assumptions that, followed to their logical conclusion, reveals a verification gap that cannot be audited, cannot be programmatically detected, and is available to any GitHub user with a free account.

---

## The Documented Chain

### 1. GitHub promises trust through verification

GitHub docs state that commit signature verification lets other people "be confident that the changes come from a trusted source."

> [Managing commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification)

### 2. Verification checks the committer's registered key

Verification checks whether the commit is signed with a GPG/SSH key registered to a GitHub account.

> [About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification)

### 3. Git has two identity fields per commit

Every commit has an **author** (who wrote the code) and a **committer** (who applied it). Both are set freely via environment variables — `GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`, `GIT_COMMITTER_NAME`, `GIT_COMMITTER_EMAIL`.

> [Git Internals — Environment Variables](https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables)

### 4. GitHub displays the author, hides the committer

GitHub's UI displays the **author** prominently — name, avatar, profile link. The committer is hidden behind a secondary click. The green "Verified" badge sits next to the author's name.

### 5. Verification binds only to the committer's key

The author field is not verified, not validated, and not constrained in any way. The API exposes this directly — `author`, `committer`, and `verification` are separate objects on every commit.

> [REST API — Git Commits](https://docs.github.com/en/rest/git/commits)

---

## The Logic Flaw

The badge says **"Verified"** next to the author's name — but it verified the **committer's** key. These can be two completely different people.

GitHub's own API confirms this: a commit can return:

```
author: torvalds
committer: rhCat
verification.verified: true
verification.reason: valid
```

The UI shows **Linus Torvalds** with a green checkmark. The signing key is mine.

This is not a bug in the crypto. The GPG signature is valid. The flaw is in what "Verified" communicates versus what it actually checks.

---

## GitHub Knows — And Gated the Defense Behind the Victim

GitHub has a **"Partially verified"** badge state. It triggers when author ≠ committer **and the author has enabled vigilant mode**.

> [Displaying verification statuses for all of your commits](https://docs.github.com/en/authentication/managing-commit-signature-verification/displaying-verification-statuses-for-all-of-your-commits)

This means GitHub is aware that author-committer mismatch is a trust problem. But the defense is:

- **Opt-in** — off by default
- **Gated on the impersonated user's account** — not the attacker's
- **The attacker controls whether it fires** by choosing victims who haven't enabled vigilant mode

Linus Torvalds hasn't enabled it. Neither have most GitHub users.

### Invisible to Machines

The `verification.reason` field in the API has **15 possible values** (`valid`, `unsigned`, `bad_email`, `unknown_key`, `expired_key`, `not_signing_key`, `gpgverify_error`, `gpgverify_unavailable`, `unknown_signature_type`, `no_user`, `unverified_email`, `malformed_signature`, `invalid`, `bad_cert`, `ocsp_pending`).

**None of them indicate author-committer mismatch.**

The "Partially verified" state exists only in the UI layer. Automated consumers — GitHub Actions, CI/CD pipelines, security tools — query the API, where `verification.verified=true` and `verification.reason=valid` is all they see. The mismatch is invisible to machines.

---

## Why This Cannot Be Fixed Downstream

### No audit trail

GitHub does not log or notify when someone authors a commit under another user's identity. The impersonated user has no way to know.

### No programmatic detection

The `Co-Authored-By` trailer — which GitHub renders as a first-class attribution with avatar and profile link — is completely unverified. Anyone can add any GitHub user as a co-author. No notification, no consent, no API to query "commits attributed to me that I didn't make."

### Verification persists after key removal

GitHub documents this as intended: once a commit is verified within a repository's network, it stays verified even if the signing key is later removed from the account. You cannot revoke trust retroactively.

### No author-committer binding exists

No GitHub API endpoint, webhook payload, or Action context exposes whether the author field was verified. There is no field, no flag, no event. Defenders cannot build this check because the data doesn't exist.

---

## Why This Is Available to Anyone

Setting `GIT_AUTHOR_NAME` and `GIT_AUTHOR_EMAIL` requires no special access. Any GitHub user with a free account and a registered GPG key can produce "Verified" commits attributed to any other user. The committer identity (the only verified field) is visually suppressed in GitHub's UI.

This is not a privilege escalation — it is the default behavior.

---

## Why This Matters Now

### Recent supply chain precedent

The **tj-actions/changed-files** supply chain compromise (March 2025) demonstrated that CI/CD pipelines are active attack surfaces.

> [CISA Advisory — CVE-2025-30066](https://www.cisa.gov/news-events/alerts/2025/03/18/supply-chain-compromise-third-party-tj-actionschanged-files-cve-2025-30066-and-reviewdogaction)

The **Shai-Hulud** npm worm (September 2025) showed that trust-chain assumptions across package registries propagate compromise at scale.

Both attacks exploited the gap between what automation *trusts* and what is actually *verified*.

### 1Password check-signed-commits-action — 167 consumers

[1Password's `check-signed-commits-action`](https://github.com/marketplace/actions/check-signed-commits-in-pr) (167 consumers including Electron, Canonical, and FABRIC research infrastructure) checks `verification.verified` via the API — where author-committer mismatch is invisible. Spoofed commits pass with a green checkmark.

I confirmed this empirically across five POC pull requests.

Electron's security team triaged this and confirmed the action is **decorative** in their architecture — they enforce commit signing via GitHub Rulesets and custom checks, not the 1Password action. But 167 other consumers may rely on it as a real security gate.

### SLSA delegates to the platform

SLSA Source Track explicitly delegates source integrity to the hosting platform. If the platform's "Verified" means "someone signed this, but not necessarily the person displayed," the entire provenance chain has a gap at its foundation.

> [SLSA Source Requirements (Draft)](https://slsa.dev/spec/draft/source-requirements)

### Namespace governance gap

I scanned 64 AI-agent username variants on GitHub — **62% are squatted** by unaffiliated individuals, **16% are unregistered**. `@chatgpt` and `@copilot` are unregistered. Anyone can register `claude-agent`, push "Verified" commits, and visually blend with legitimate bot activity in repositories that use AI-assisted workflows.

Combine this with the trend toward AI agents committing code through CI — where every merged PR is trusted because the badge is green and the API says `verified=true` — and the attack surface is not theoretical.

---

## Affected Ecosystem

| Category | Details |
|----------|---------|
| **HIGH RISK consumers** | 25 repos using the action as upstream gate with post-merge auto-propagation conditions |
| **MEDIUM RISK consumers** | 72 repos using the action in CI with manual publish workflows |
| **LOW RISK consumers** | 45 repos using the action as advisory/decorative check |
| **Total affected** | 167 consumers of 1Password/check-signed-commits-action |

Detailed affected-consumer list with exact workflow deep links will be published after coordinated disclosure window closes.

---

## CWEs

- **CWE-345**: Insufficient Verification of Data Authenticity
- **CWE-290**: Authentication Bypass by Spoofing

---

## Disclosure Timeline

| Date | Action |
|------|--------|
| 2025-05 | Initial discovery and POC development |
| 2025-05 | HackerOne submission — GitHub |
| 2025-05 | MSRC case opened — Electron/Teams federal path |
| 2025-05 | CERT/CC VINCE case submitted |
| 2025-05 | Electron Security Advisory → triaged, confirmed non-issue for their architecture |
| 2025-05 | 1Password action notification pending |
| 2025-05 | VINCE 5 calendar days without response (3-day SLA exceeded) |
| 2025-05 | Public disclosure |

---

## What Should Change

1. **GitHub**: Bind the Verified badge to an author=committer match, or show a distinct badge state when author ≠ committer regardless of vigilant mode, or surface the committer identity with equal prominence.

2. **GitHub API**: Expose author-committer binding status in the `verification` object so automated tools can detect mismatch.

3. **1Password**: Update `check-signed-commits-action` to compare author and committer fields, not just `verification.verified`.

4. **Downstream consumers**: Use GitHub Rulesets with required signed commits as the enforcement mechanism (the "Electron pattern"), not CI actions that query the API.

---

## Contact

Rui He — [ruihe93@gmail.com](mailto:ruihe93@gmail.com)

Full findings repository will be made public after coordinated disclosure window closes.
