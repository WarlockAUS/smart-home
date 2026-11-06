# Blueprint Update Checker (v1.4)

A generic Home Assistant blueprint that monitors any GitHub-hosted
blueprint for new releases and notifies you when the author's version
has moved ahead of the version you last saw. Built to cover the gap
left by blueprints that aren't distributed via HACS, where Home
Assistant has no native "update available" notification.

## How it works

Once a day (and once on Home Assistant restart), the automation
fetches the raw YAML file of a tracked blueprint from GitHub,
extracts a version string from it using a regex, and compares it
against the last version you recorded in an `input_text` helper. If
the live version differs from your stored version, a persistent
notification appears in the Home Assistant frontend (and an optional
mobile push, if enabled), and the helper is updated to the new value.
If they match, nothing happens — no noise, no duplicate alerts.

## Scope and Limitations

This checker relies on the tracked blueprint stating its version
number in the top-level `name:` field (e.g. `Some Blueprint v1.2` or
`Some Blueprint (v2026.11.13)`). There is no official version field
in the Home Assistant blueprint schema — this is purely a community
convention, applied inconsistently across authors.

Deliberately out of scope: attempting to detect version numbers
written elsewhere in a blueprint file (e.g. free text inside the
`description:` block, bold Markdown, changelogs). Earlier drafts of
this checker experimented with fallback regex patterns for this, but
it was dropped in favour of a single, well-tested pattern — chasing
every author's personal formatting convention produces brittle
regex and a growing maintenance burden for diminishing benefit.

**Practical guidance:** if a blueprint you want to track doesn't
state its version in the `name:` field, this checker won't reliably
detect updates to it. In that case, periodically re-importing the
blueprint manually (which always pulls the latest content regardless
of version text) is the simpler and more honest approach.

**Confirmed working examples:**
- This blueprint itself (`name: Blueprint Update Checker v1.4`)
- [NSPanel Easy Configuration](https://github.com/edwardtfn/NSPanel-Easy) (`name: NSPanel Easy Configuration (v2026.11.13)`)

**Not currently supported:**
- Blackshome's Low Battery Notifications and Actions blueprint, which
  states its version separately from the `name:` field. Track this
  one via periodic manual re-import instead.

## Prerequisites

### 1. One-time `rest_command` setup

Add the following to your `configuration.yaml`. This only needs to
be done once, regardless of how many blueprints you track afterwards.

```yaml
rest_command:
  fetch_raw_text:
    url: "{{ url }}"
    method: GET
