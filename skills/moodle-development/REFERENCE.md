# Moodle Development Reference

## Official documentation

- Getting started: <https://moodledev.io/general/development/gettingstarted>
- Plugin types: <https://moodledev.io/docs/apis/plugintypes>
- DML/Data manipulation API: <https://moodledev.io/docs/apis/core/dml>
- Coding style: <https://moodledev.io/general/development/policies/codingstyle>
- PHPUnit: <https://moodledev.io/general/development/tools/phpunit>
- Behat: <https://moodledev.io/general/development/tools/behat>
- Accessibility: <https://moodledev.io/general/development/policies/accessibility>
- Security: <https://moodledev.io/general/development/policies/security>

When docs and local code disagree, prefer the docs for the target Moodle branch plus established local patterns.

## Component and file conventions

Typical plugin layout:

```text
pluginroot/
├── version.php              # component, version, requires, maturity/release
├── lib.php                  # callbacks/hooks only when needed
├── settings.php             # admin settings
├── db/
│   ├── install.xml          # XMLDB schema for fresh install
│   ├── upgrade.php          # schema/data upgrades
│   ├── access.php           # capabilities
│   ├── events.php           # observers
│   ├── services.php         # web service definitions
│   └── tasks.php            # scheduled tasks
├── classes/
│   ├── form/                # namespaced forms where appropriate
│   ├── output/              # renderables, exporters
│   ├── event/               # event classes
│   ├── external/            # external API classes
│   ├── task/                # scheduled/ad-hoc tasks
│   └── privacy/provider.php # GDPR provider if storing user data
├── lang/en/{component}.php  # strings
├── templates/*.mustache     # HTML templates
├── amd/src/*.js             # source JS
├── amd/build/*.min.js       # built JS, usually committed in Moodle plugins
└── tests/                   # PHPUnit/Behat tests
```

Frankenstyle component names are mandatory in strings, capabilities, events, and privacy APIs: `plugintype_pluginname`.

## Security rules

- Never use superglobals directly. Use `required_param`, `optional_param`, `required_param_array`, `optional_param_array` with strict `PARAM_*` types.
- All access-controlled pages need `require_login()` and context-aware capability checks.
- Hiding buttons is not authorization. Re-check permissions server-side.
- All state-changing requests need sesskey protection (`require_sesskey()` or `moodleform` validation).
- Use `$DB` placeholders for all dynamic SQL values. Do not interpolate variables into SQL.
- Escape output according to context:
  - Plain text: `s($value)`.
  - Moodle formatted text: `format_text()` with trusted flags chosen deliberately.
  - Names/titles: `format_string()`.
  - URLs: `moodle_url` and rendered attributes, not manual string HTML.
- File handling should use Moodle File API and draft areas, not arbitrary writable paths.
- If storing personal data, implement privacy provider methods and export/delete behaviour.

## Database and upgrade decisions

Use XMLDB for schema. A correct schema change usually includes:

1. `db/install.xml` updated for new installs.
2. `db/upgrade.php` upgrade step for existing installs.
3. `version.php` version bump.
4. Tests or manual upgrade verification.

Upgrade step pattern:

```php
function xmldb_local_example_upgrade($oldversion) {
    global $DB;
    $dbman = $DB->get_manager();

    if ($oldversion < 2026050900) {
        $table = new xmldb_table('local_example_items');
        $field = new xmldb_field('enabled', XMLDB_TYPE_INTEGER, '1', null, XMLDB_NOTNULL, null, '1');

        if (!$dbman->field_exists($table, $field)) {
            $dbman->add_field($table, $field);
        }

        upgrade_plugin_savepoint(true, 2026050900, 'local', 'example');
    }

    return true;
}
```

Prefer idempotent, simple upgrade steps. Do not rely on request/session state in upgrades.

## Data API choices

- CRUD: `$DB->get_record`, `get_records`, `insert_record`, `update_record`, `delete_records`.
- Complex reads: `$DB->get_records_sql($sql, $params)` with named placeholders.
- Transactions: `$transaction = $DB->start_delegated_transaction(); ... $transaction->allow_commit();`.
- Time: use `time()` for timestamps unless local code has a time provider.
- Large data sets: use recordsets and close them.

## Forms, pages, and output

- Use `moodleform` for forms; it handles sesskey and validation patterns.
- Use `$PAGE->set_context()`, `set_url()`, `set_title()`, `set_heading()` before output.
- Prefer renderers/renderables/exporters and Mustache for complex UI.
- JavaScript should be AMD/ES module per target Moodle branch; build with Grunt when required.
- User-facing strings always go through `get_string()`.

## Capabilities and contexts

Capability names use `component:capability`, e.g. `local_example:manage`.

Common contexts:

- System-wide admin/config: `context_system::instance()`.
- Course-level feature: `context_course::instance($courseid)`.
- Activity module: `context_module::instance($cmid)` plus `get_coursemodule_from_id()`.
- User-specific data: `context_user::instance($userid)` only when semantically correct.

Define capabilities in `db/access.php`; then gate server-side code with `require_capability()` for hard requirements or `has_capability()` for conditional UI/logic.

## Events and tasks

Use events when other parts of Moodle may need to react or logs need meaningful audit records. Event classes live in `classes/event` and should include context, objectid, relateduserid where relevant.

Use scheduled tasks for periodic jobs, ad-hoc tasks for background work triggered by a user action. Keep tasks resumable and permission-safe.

## Testing strategy

- PHPUnit for services, DB logic, capability-sensitive methods, events, tasks, privacy provider.
- Behat for workflows visible to users/admins.
- Generators: use Moodle test data generators rather than hand-inserting all records.
- Reset state through Moodle test helpers; avoid brittle dependence on global data.
- For bug fixes, reproduce with a failing test first when feasible.

## Code review heuristics

Reject or revise changes that introduce:

- Framework-generic code bypassing Moodle APIs.
- New DB columns without install.xml + upgrade.php + version bump.
- Raw HTML echoing mixed with business logic.
- Hardcoded strings or untranslated UI.
- Permission checks only in templates/JS.
- SQL string concatenation with variables.
- Personal data storage without privacy API consideration.
- New UI flows without Behat when behaviour is important.
- Use of APIs unavailable in the target Moodle branch.
