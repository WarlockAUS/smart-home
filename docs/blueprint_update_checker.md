# Blueprint Update Checker (v1.6)

A generic Home Assistant blueprint that monitors any GitHub-hosted
blueprint for new releases and notifies you when the author's version
has moved ahead of the version you last saw. Built to cover the gap
left by blueprints that aren't distributed via HACS, where Home
Assistant has no native "update available" notification.

## How it works

Once a day (and once on Home Assistant restart), the automation
fetches the raw YAML file of a tracked blueprint from GitHub, and
tries up to three regex patterns in order to extract a version
string: a primary pattern anchored to the blueprint's top-level
`name:` field, and two fallback patterns covering common "Version"
text conventions seen in some blueprint descriptions. The first
pattern that finds a match wins. That value is compared against the
last version you recorded in an `input_text` helper. If the live
version differs from your stored version, you're notified via
whichever channel(s) are enabled, and the helper is updated to the
new value. If they match, nothing happens — no noise, no duplicate
alerts.

## Scope and Limitations

There is no official version field in the Home Assistant blueprint
schema — versioning is purely a community convention, applied
inconsistently across authors. This checker deliberately supports
only the specific conventions it has been tested against, rather than
attempting a fully generic pattern that tries to catch every possible
style:

1. **Primary** — version embedded in the top-level `name:` field,
   e.g. `name: Some Blueprint v1.4` or
   `name: Some Blueprint (v2026.11.13)`.
2. **Fallback 1** — a `**Version: X.X**` style line (asterisks
   wrapping the whole phrase, including the colon).
3. **Fallback 2** — a `**Version**: X.X` style line (asterisks
   wrapping only the word "Version", colon outside).

Deliberately out of scope: any other convention not listed above —
including plain unbolded "Version:" text, version numbers embedded
in free-flowing prose, or changelog-style entries. Earlier drafts
experimented with broader, more generic patterns intended to catch
any possible phrasing, but this was dropped in favour of a small,
precisely-tested set — chasing every author's personal formatting
convention produces brittle regex and a growing maintenance burden
for diminishing benefit. Each additional pattern here was added only
after being confirmed against a real, specific file.

**Practical guidance:** if a blueprint you want to track doesn't
match any of the three supported conventions, this checker won't
reliably detect updates to it. In that case, periodically re-importing
the blueprint manually (which always pulls the latest content
regardless of version text) is the simpler and more honest approach.

**Confirmed working examples:**
- This blueprint itself (`name: Blueprint Update Checker v1.6`) — Primary pattern
- [NSPanel Easy Configuration](https://github.com/edwardtfn/NSPanel-Easy) (`name: NSPanel Easy Configuration (v2026.11.13)`) — Primary pattern
- A blueprint using `**Version**: 5.5.9` style — Fallback 2 pattern

**Not currently supported:**
- Blackshome's Low Battery Notifications and Actions blueprint —
  exact version convention not yet confirmed against the three
  supported patterns. Track via periodic manual re-import instead.

## Prerequisites

### 1. One-time `rest_command` setup

Add the following to your `configuration.yaml`. This only needs to
be done once, regardless of how many blueprints you track afterwards.

```yaml
rest_command:
  fetch_raw_text:
    url: "{{ url }}"
    method: GET
