# Blueprint Update Checker (v1.2)

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
the live version differs from your stored version, you get a
notification and the helper is updated to the new value. If they
match, nothing happens — no noise, no duplicate alerts.

## Prerequisites

### 1. One-time `rest_command` setup

Add the following to your `configuration.yaml`. This only needs to
be done once, regardless of how many blueprints you track afterwards.

```yaml
rest_command:
  fetch_raw_text:
    url: "{{ url }}"
    method: GET
