---
title: OpenClaw LTS Release Policy
authors:
  - Kevin Lin
created: 2026-05-27
last_updated: 2026-05-27
rfc_pr: TBD
---

# Proposal: OpenClaw LTS Release Policy

## Summary

Add an opt-in OpenClaw LTS channel by designating selected already-shipped
stable releases for longer support. Daily `stable` releases remain the default
install and update path, while `lts` becomes a first-class package and update
target backed by a fixed active LTS branch, the npm `lts` dist-tag, explicit
support dates, narrow backport rules, and documented plugin compatibility.

## Motivation

OpenClaw's daily stable cadence gives users fast access to fixes and
improvements, but some operators need a lower-churn supported line for
production or enterprise deployments. Those users need predictable install
commands, a bounded support window, explicit backport criteria, and a clear
answer for whether core packages and plugins remain compatible.

The LTS policy should provide that stability without creating a second default
release train. The daily stable release remains the proving ground. Maintainers
select a stable release for LTS only after it has shipped, soaked, and produced
the normal release evidence.

## Goals

- Preserve daily `stable` as the default install and update target.
- Add `lts` as an explicit opt-in channel for package installs and
  `openclaw update`.
- Designate LTS from already-shipped stable releases instead of building a
  separate release factory.
- Reuse existing stable correction versions such as `YYYY.M.D-N`.
- Maintain one active LTS line by default.
- Define the LTS support window, eligible backports, validation bar, docs
  obligations, and end-of-life behavior.
- Make bundled and explicitly-covered official plugin compatibility part of
  the support contract.
- Fail closed when no active LTS artifact exists.

## Non-Goals

- Changing the daily stable release cadence.
- Making LTS the default install target.
- Introducing a new `-lts` version suffix or version family.
- Guaranteeing support for every external plugin.
- Backporting normal feature work, refactors, or broad compatibility churn.
- Blocking daily stable releases on LTS maintenance work.
- Defining a multi-line LTS program before maintainers intentionally expand
  support capacity.

## Proposal

### Policy model

OpenClaw LTS is a support designation applied to a selected stable release after
that release ships through the normal release process. LTS is additive: it does
not replace `stable`, `beta`, or `dev`, and it does not slow the daily stable
train.

The initial policy is:

- `stable` remains the default user-facing version and continues to track the
  newest daily stable release.
- Maintainers may designate an already-shipped stable release as the active LTS
  base.
- LTS maintenance releases keep the existing stable correction format
  `YYYY.M.D-N`.
- OpenClaw supports at most one active LTS line at a time by default.
- Users opt in through `openclaw update --channel lts`, `openclaw@lts`, or an
  installer option that resolves to the same LTS package target.

### LTS selection

Maintainers may designate a stable release as LTS only when all of the following
are true:

- The release already shipped as a normal stable release through the existing
  release flow.
- The release has full release evidence, including the release checklist and a
  successful `Full Release Validation` run.
- The release has a documented upgrade and rollback story that does not require
  operator-only recovery for normal install or update paths.
- The release has a published compatibility coverage table for bundled plugins
  and any official external plugins covered by the LTS promise.
- The release has soaked long enough in the stable population for maintainers to
  judge it lower risk than the moving daily tip.

This makes LTS a promotion decision on a stable artifact, not a parallel
artifact-build process.

### Support window

Each LTS line receives six months of active support from designation. The final
month is maintenance-only: new backports are limited to security fixes and
critical install, update, or upgrade blockers.

At end of life, users should move to either the current daily `stable` line or
the next designated LTS line. Before the active LTS line reaches end of life,
maintainers must retarget the active LTS selector to the next supported line.

### Eligible backports

LTS backports are intentionally narrow. Eligible changes are:

- Security fixes.
- Dependency or packaging fixes required to keep installation, update, signing,
  notarization, or registry publication working.
- Critical regressions in core agent, gateway, update, or packaged install
  flows.
- Upgrade blockers from the base LTS release or an earlier correction in the
  same LTS line.
- Bundled-plugin or explicitly-supported official-plugin compatibility fixes.
- Release-blocking docs or operator guidance corrections where the current docs
  would cause normal install, upgrade, or recovery paths to fail.

The following changes are not eligible by default:

- New features.
- New default behavior.
- Normal refactors.
- New provider, channel, plugin, or product surfaces.
- Broad compatibility churn that is not required to preserve the LTS contract.

Fixes land on `main` first, then are cherry-picked to the maintained LTS branch.

### Branch, tag, and publish mechanics

The normal daily stable process continues to cut `release/YYYY.M.D` branches
from `main` and publish stable releases. LTS designation adds active-channel
metadata and a maintenance branch around an already-published release.

The active LTS branch model is:

- Maintainers create `release/lts` from the selected stable release line.
- `release/lts` is the canonical git-side selector for the active LTS line.
- LTS corrections use existing stable correction tags such as `vYYYY.M.D-N`.
- `main` continues to move independently and is never gated by LTS maintenance.
- When an LTS line retires, maintainers retarget `release/lts` to the next
  active LTS line and may preserve the retired line as
  `release/lts-{version}` for history.

The package publication model is:

- The initial LTS designation moves or adds npm dist-tag `lts` to the exact
  already-published stable version.
- Designation does not rebuild or republish different bytes for the base
  stable release.
- Later LTS correction releases move npm dist-tag `lts` forward to the newest
  supported correction in the same line.
- Token-bearing dist-tag mutation should reuse the existing private dist-tag
  workflow and operator path.
- Active LTS docs and compatibility tables are updated in the same designation
  or correction window as the dist-tag move.

### Install and update semantics

`stable` remains the default channel and resolves to the newest daily stable
release. `lts` is explicit and resolves to the active supported LTS line.

The package-manager path for a new LTS install is:

```bash
npm install -g openclaw@lts
openclaw update --channel lts
```

The installer path should expose the same target:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm --version lts
openclaw update --channel lts
```

The existing-install path is:

```bash
openclaw update --channel lts
```

If the current install is newer than the active LTS line, switching to LTS may
be a downgrade. OpenClaw should surface that clearly using the existing
downgrade-prompt behavior.

Resolution rules:

- `openclaw update --channel stable` targets the newest daily stable release.
- `openclaw update --channel lts` targets the active supported LTS line.
- npm installs on the LTS channel resolve through npm dist-tag `lts`.
- git installs on the LTS channel follow `release/lts` and select the newest
  stable or stable-correction tag reachable from that branch.
- Exact LTS versions and tags remain supported for pinned installs and
  diagnostics.
- Update status should distinguish configured channel from resolved artifact.
- Persisted `update.channel = lts` should reuse stable-class auto-update delay
  and jitter behavior.

LTS resolution must fail closed when no active LTS artifact exists. It must not
silently fall back to `latest`.

### Version identity

OpenClaw should express LTS through channel and support metadata, not through a
new version suffix.

- Base LTS releases keep the normal stable form `YYYY.M.D`.
- LTS corrections keep the existing stable correction form `YYYY.M.D-N`.
- `lts` is represented by update-channel and dist-tag selection.

Numeric suffixes such as `-1` already represent stable corrections in the
OpenClaw release contract. Named suffixes such as `-beta.2` represent
prereleases. A new `-lts.N` shape would expand OpenClaw's custom version
semantics while looking like a prerelease to generic SemVer tooling.

### Plugin support

The LTS promise must name its plugin scope.

- Bundled `@openclaw/*` plugins that ship with the LTS core artifact are covered
  by the same correction policy.
- First-party `@openclaw/*` plugins bundled in normal OpenClaw builds follow the
  LTS policy as part of the LTS artifact, even when they also exist as separate
  fallback packages.
- Official external plugins are covered only when the active LTS docs list them
  in the plugin LTS table.
- Unpinned `default` or `latest` plugin specs are outside the LTS contract
  unless OpenClaw resolves them through the plugin LTS table.
- Untrusted third-party plugins remain user-managed and are never implicitly
  rewritten onto an LTS-specific target.

When `openclaw update --channel lts` encounters a tracked official external
plugin listed in the active LTS table, OpenClaw should resolve that plugin by
deriving the same-version npm or ClawHub spec from catalog metadata. Exact
pinned external plugin specs are not rewritten unless the table names an
explicit LTS target.

When a tracked official external plugin is not listed in the active LTS table,
OpenClaw should complete the core LTS update, leave the plugin unchanged, and
emit a structured warning that the plugin is outside the LTS support promise.

### Plugin LTS table

The first LTS release should use a checked-in Markdown table as the plugin
support source of truth. The table answers "which plugins are under LTS?" The
existing `YYYY.M.D` and `YYYY.M.D-N` version scheme answers "which version?"

Covered external plugins use the same version as the active LTS core by
default. For example, active core LTS `openclaw@2026.6.10-1` maps a covered
WhatsApp plugin to `@openclaw/whatsapp@2026.6.10-1` or
`clawhub:@openclaw/whatsapp@2026.6.10-1`.

The active LTS docs should publish a table in this shape:

| Plugin | Package or install spec | LTS status | LTS version | Notes |
| --- | --- | --- | --- | --- |
| Bundled `@openclaw/*` plugins | Included in `openclaw` | Covered | Same as `openclaw@lts` | Covered by core release validation. |
| `whatsapp` | `@openclaw/whatsapp` or `clawhub:@openclaw/whatsapp` | Covered | Same as `openclaw@lts` | Example official external plugin once package acceptance passes. |
| Any official external plugin not listed above | Current user install | Not covered | Unchanged | `openclaw update --channel lts` warns and leaves it pinned. |

The table should be deliberately small for the first LTS line. Maintainers can
add rows as official external plugins gain release proof. JSON or a dedicated
compatibility registry can wait until the table becomes hard to maintain.

Table validation must prove:

- every covered plugin row matches a bundled plugin or official external package
  in the generated plugin inventory
- every covered official external plugin has package acceptance proof
- every same-version derived npm or ClawHub spec is syntactically valid
- no covered target uses `latest`, `default`, or an unversioned package spec

The structured warning should include `pluginId`, `packageName`, current install
spec or version when known, active LTS version, and a stable reason code such as
`outside_lts_contract`.

### Validation bar

Before designating a release as LTS, OpenClaw should have:

- Full release checklist evidence from the existing stable release flow.
- `Full Release Validation` at `release_profile=full`.
- Package acceptance against the exact published version.
- Cross-OS packaged install and upgrade proof.
- Upgrade proof from the base LTS release to the latest correction in that
  line, when a correction exists.
- Plugin compatibility proof for bundled plugins and any explicitly-covered
  official plugins.
- A validated Markdown plugin LTS table for the active line.
- Published docs that name the active LTS version, support window, install
  commands, compatibility scope, and end-of-life date.

For LTS correction releases, maintainers may reuse previous umbrella evidence
and rerun the smallest affected release-validation box, but each correction
still needs exact-version package proof before announcement.

### Documentation and communication

OpenClaw must publish and keep current:

- The active LTS version.
- Support start and end dates.
- Exact install and update commands.
- The plugin LTS table.
- The structured warning users see when a tracked plugin is outside the LTS
  contract.
- Which fixes qualify for backport consideration.
- The rule that daily `stable` remains the default recommendation.

### Rollout plan

1. Adopt this policy and choose the first LTS candidate release.
2. Add the active LTS branch, tag, validation, compatibility, and docs
   obligations to the release runbook.
3. Add npm `lts` dist-tag promotion and `openclaw update --channel lts`.
4. Add fail-closed LTS resolution for package and git install paths.
5. Add the Markdown plugin LTS table, lightweight validation, and updater
   warnings for official external plugins that are not listed.

### Worked example

Assume:

- The newest daily stable is `2026.6.18`.
- The active LTS base release is `2026.6.10`.
- The newest LTS correction is `2026.6.10-1`.

Then expected resolution is:

| Action | Result |
| --- | --- |
| `npm install -g openclaw@latest` | Installs `2026.6.18`. |
| `npm install -g openclaw@lts` | Installs `2026.6.10-1`. |
| `openclaw update --channel stable` | Resolves to `2026.6.18`. |
| `openclaw update --channel lts` | Resolves to `2026.6.10-1`. |

Daily stable users keep following the moving stable train. LTS users follow the
maintained LTS line. Version numbers remain consistent with the existing stable
and stable-correction scheme.

## Rationale

### Channel designation instead of an `-lts` suffix

LTS is a support contract, not a new artifact identity. Channel designation
preserves the existing `YYYY.M.D` and `YYYY.M.D-N` release shapes, keeps generic
SemVer tooling behavior predictable, and avoids teaching OpenClaw that named
suffixes can be stable support releases.

### Promotion after stable instead of a parallel release train

Selecting an already-shipped stable release lets the daily train remain the
quality gate and soak period. A parallel LTS factory would add build and release
complexity while reducing confidence that LTS artifacts match what users already
validated in stable.

### One active LTS line by default

One active LTS line keeps support, validation, docs, and plugin compatibility
work bounded. Multiple active LTS lines can be added later only if maintainers
have the release capacity and automation to keep each line healthy.

### Same-version plugin compatibility instead of `latest`

Core LTS stability is undermined if plugin resolution keeps following moving
`latest` or `default` specs. Same-version plugin resolution keeps covered
official plugins aligned with the active LTS core line, while the Markdown table
makes the supported plugin set obvious.

### Markdown table instead of a compatibility registry

The existing plugin compatibility registry tracks plugin-facing contract
adapters and deprecation windows. The first LTS release only needs to tell users
which plugins are included in the support promise. A Markdown table is enough for
that, and it keeps package-support state out of the API migration registry.

### Fail-closed resolution instead of falling back to stable

Falling back from `lts` to `latest` would silently erase the support promise.
Failing closed forces maintainers to fix designation metadata or publish a clear
message before users are moved onto a different release lane.

## Unresolved questions

- Which already-shipped stable release should be the first LTS candidate?
- Which docs surface should become the canonical active-LTS landing page?
- Which official external plugins should be included in the first LTS table?
