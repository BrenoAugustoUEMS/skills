---
name: moodle-development
description: Moodle dev guardrails: plugins, core APIs, security, DB upgrades, UI, tests. Use for Moodle PHP/JS, plugins, LMS activities/blocks/local plugins, gradebook, capabilities, Behat, PHPUnit.
---

# Moodle dev

Moodle first. Avoid generic PHP patterns when Moodle API exists. Before change: find Moodle version, component (`type_name`), plugin type, local conventions.

## Must-do

- Bootstrap pages with `config.php`.
- Params via `required_param()` / `optional_param()` + `PARAM_*`. Never raw `$_GET/$_POST`.
- Auth early: `require_login()`, correct `context_*`, `require_capability()` / `has_capability()`.
- Mutations need `require_sesskey()` or `moodleform` validation.
- DB via global `$DB`; placeholders only. No concatenated SQL.
- Schema: `db/install.xml` + guarded `db/upgrade.php` + `version.php` bump.
- Output: `$PAGE`, `$OUTPUT`, renderers/templates. Avoid echoed HTML.
- Escape: `s()`, `format_string()`, `format_text()`, Mustache autoescape.
- Strings: `lang/en/{component}.php` + `get_string()`.
- User data: privacy provider.
- Tests: PHPUnit for logic/DB; Behat for UI workflow.

## Files

- Capabilities: `db/access.php`
- Events: `classes/event/*`, observers `db/events.php`
- Tasks: `classes/task/*`, `db/tasks.php`
- Web services: `classes/external/*`, `db/services.php`
- UI: `classes/output/*`, `templates/*.mustache`, `amd/src/*`
- Tests: `tests/*_test.php`, `tests/behat/*.feature`

## Review smell list

Flag: superglobals, PDO/mysqli, SQL interpolation, missing login/capability/sesskey, hardcoded English, unescaped output, raw HTML, schema change without upgrade/version bump, stored user data without privacy API, UI change without tests.

## Commands

Prefer repo README/CI. Common:

```bash
vendor/bin/phpunit path/to/test.php
vendor/bin/phpcs --standard=moodle path/to/plugin
npx grunt amd
php admin/cli/upgrade.php
php admin/tool/phpunit/cli/init.php
```

Use `REFERENCE.md` only when exact API/file pattern needed.
