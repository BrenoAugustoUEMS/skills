---
name: moodle-development
description: "Guides correct Moodle core and plugin development decisions: APIs, plugin structure, security, database upgrades, UI, tests, and coding standards. Use when editing or reviewing Moodle PHP/JS code, creating Moodle plugins, writing upgrades, using Moodle APIs, or when the user mentions Moodle, LMS plugins, activities, blocks, local plugins, enrolment, gradebook, capabilities, Behat, or PHPUnit in a Moodle codebase."
---

# Moodle Development

## Prime directive

Moodle is an API-heavy PHP application. Prefer Moodle APIs and conventions over framework-generic PHP patterns. Before implementing, identify the Moodle version and plugin/component type, then preserve backwards compatibility for the supported branch.

Official docs entrypoint: <https://moodledev.io/general/development/gettingstarted>

## Decision checklist

1. **Locate the component**
   - Plugin frankenstyle: `{plugintype}_{pluginname}` (for example `mod_forum`, `local_myplugin`).
   - Component files usually live under `classes/`, `db/`, `lang/en/`, `tests/`, `templates/`, `amd/src/`.
   - Check existing code style and branch Moodle version before introducing newer APIs.
2. **Use Moodle request/page conventions**
   - Entry scripts start with `require(__DIR__ . '/../../config.php')` adjusted to depth.
   - Read input with `required_param()`, `optional_param()`, `PARAM_*` only.
   - Use `moodle_url`, `$PAGE`, `$OUTPUT`, renderer/templates; avoid raw echoed HTML.
3. **Enforce access and CSRF**
   - Call `require_login()` or course/module-specific login early.
   - Resolve `context_*` and check `require_capability()` / `has_capability()`.
   - Mutating actions require `require_sesskey()` or form validation.
4. **Use Moodle data APIs**
   - Use global `$DB`; never concatenate SQL values. Use placeholders and params.
   - Schema changes go in `db/install.xml` and `db/upgrade.php` via XMLDB manager.
   - Bump `$plugin->version` in `version.php` for upgrade steps.
5. **Render safely**
   - Escape user text with `s()`, `format_string()`, `format_text()`, or Mustache autoescaping.
   - Use renderables, output classes, and Mustache templates for non-trivial UI.
6. **Internationalise everything user-facing**
   - Put strings in `lang/en/{component}.php` and fetch with `get_string()`.
7. **Choose the right extension point**
   - Capabilities: `db/access.php`.
   - Events: `classes/event/*` and observers in `db/events.php`.
   - Scheduled/ad-hoc tasks: `classes/task/*` and `db/tasks.php`.
   - Web services/external API: `classes/external/*`, `db/services.php`.
   - Privacy/GDPR: implement provider interfaces in `classes/privacy/provider.php` when storing user data.
8. **Test with Moodle tooling**
   - Add PHPUnit tests under `tests/*_test.php` using Moodle generators.
   - Add Behat scenarios for user workflows when UI behaviour matters.
   - Run targeted tests and Moodle codechecker before finishing.

## Common workflows

### Add a plugin feature

- Confirm plugin type and supported Moodle versions.
- Find existing component patterns: forms, renderers, services, capabilities, events, tests.
- Add data model changes through XMLDB + upgrade step if persistent.
- Add capability checks before reads/writes, not only UI hiding.
- Add lang strings, tests, and upgrade-safe defaults.

### Add or change DB schema

- Edit `db/install.xml` for fresh installs.
- Add an idempotent upgrade step in `db/upgrade.php` guarded by `$oldversion < YYYYMMDDXX`.
- Use `$dbman = $DB->get_manager()` and XMLDB table/field/index objects.
- Bump `version.php`; never edit production schema directly.

### Add a page or action endpoint

- Bootstrap `config.php`, parse params, derive context, `require_login()`, check capability.
- For POST/delete/update, verify sesskey and redirect after success.
- Set `$PAGE->set_url()`, context, title/heading; render via `$OUTPUT`.

### Review Moodle code

Look for: raw `$_GET/$_POST`, direct PDO/mysqli, string-concatenated SQL, missing `require_login`, missing capability, missing sesskey, hardcoded English, unescaped output, schema changes without upgrade, user data without privacy provider, and UI changes without Behat/PHPUnit coverage.

## Commands to try

Use project-specific paths/config first. Common examples:

```bash
vendor/bin/phpunit path/to/plugin/tests/some_test.php
vendor/bin/phpcs --standard=moodle path/to/plugin
npm install
npx grunt amd
php admin/cli/upgrade.php
php admin/tool/phpunit/cli/init.php
```

If commands differ in the repository, follow its README/CI scripts.

## Deeper reference

Read [REFERENCE.md](REFERENCE.md) for API choices, file conventions, security rules, and links to Moodle docs.
