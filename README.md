# dropredo-tests

An open format for end-to-end browser tests written in plain English.

A `dropredo-tests/` folder in your repository describes what a careful tester would do and check, step by step: "visit the login page", "click the button with text Sign in", "the heading Dashboard is visible". There are no selectors, no XPath and no test code. An AI coding agent can write the whole folder from your codebase in a few minutes, and you can read every line of it.

[DropRedo](https://dropredo.com) runs these files in real Chromium, Firefox and WebKit browsers. It works out each step's action and element against the live page as it runs, records video, and repairs a step when the page changes under it. The format is free to write and keep; running it is what DropRedo sells.

```yaml
applications:
  acme:
    groups:
      authentication:
        tests:
          user can sign in with valid credentials:
            environments: [prod]
            steps:
              - step: visit {{ base_uri }}/login
                assertions:
                  - the heading "Sign in to Acme" is visible
              - fill the {{ user_email }} into the email address field
              - fill the {{ user_password }} into the password field
              - step: click the button with text "Sign in"
                assertions:
                  - the browser is on the dashboard page
```

## What's here

| Path | What it is |
|---|---|
| [`PROMPT.md`](PROMPT.md) | The complete authoring prompt. Give it to an AI coding agent (Claude Code, Cursor, Copilot, Codex…) inside your repository, and it writes your `dropredo-tests/` folder. It is also the full reference for the format. |
| [`schema/dropredo-tests.schema.json`](schema/dropredo-tests.schema.json) | A JSON Schema for one file, for autocomplete and checking in your editor. |
| [`limits.json`](limits.json) | The limits an import is held to. |
| [`examples/`](examples) | Real suites. `dropredo-website` is the 7-file suite DropRedo runs against its own website; `acme` is the two-file example from the prompt. |

`PROMPT.md`, the schema and `limits.json` are generated from the same code DropRedo imports with, so they never promise something the importer doesn't accept.

## Write a suite

1. Open your application's repository in your coding agent.
2. Paste [`PROMPT.md`](PROMPT.md) in as the task.
3. The agent reads your routes, templates and existing tests, writes `dropredo-tests/`, and lists every `REPLACE_ME` value (test account credentials, for example) for you to fill in.

You can also write the files by hand. The format is small:

```
dropredo-tests/
  dropredo.yaml          # applications, environments, variables — no groups
  authentication.yaml    # one file per feature area, each declaring one group
  checkout.yaml
  admin/billing.yaml     # nest when an area has several modules
```

- **Applications** hold **environments** (each with **variables**, used in steps as `{{ base_uri }}`) and **groups**. Groups hold **tests**, under a `tests:` key.
- A **test** has `environments` (where it runs; the first is where it is generated) and `steps`. It may name one test it `depends_on`; it then starts in that test's browser session, signed in.
- A **step** is one action in plain English, either as a bare string or as `step:` with `assertions:`. An **assertion** is one visible fact in plain English. `negate: true` asserts the opposite.
- Optional fields pin what you already know: a step's `action`, an assertion's `type`, `expected` or `attribute`. A step can also pause the run for a human (`pause_for_manual_action: true`, for a TOTP code or a CAPTCHA) or run in a named browser tab (`tabs:` and `tab:`).

### Editor support

Add this line to the top of each file to get autocomplete and checking in any editor that uses the YAML language server (VS Code with the Red Hat YAML extension, JetBrains IDEs, Neovim):

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/buildnextdot/dropredo-tests/main/schema/dropredo-tests.schema.json
```

### Rules the schema can't check

The schema checks one file at a time. These rules span files, and are checked when the folder is imported:

- Each environment and variable is declared once, in `dropredo.yaml`. Each test is declared once, and one group is never split across two files.
- Every variable has a non-empty value. Write `"REPLACE_ME"` until you have the real one.
- Every `{{ token }}` a test uses is declared in every environment that test runs in.
- A test names at most one test in `depends_on`, in the same application, and that test runs in every environment its dependant runs in. There are no cycles.
- Every step's `tab:` names a tab its test declares, and `use-parent-test-session-in-tabs` needs a `depends_on`.
- Test names are unique within an application.

The full list, with how to write steps and assertions that resolve reliably, is in [`PROMPT.md`](PROMPT.md).

## Run a suite

1. [Sign up for DropRedo](https://app.dropredo.com/register?utm_source=github&utm_medium=open_spec&utm_campaign=dropredo_tests) (there is a free plan), open **Import Tests** and upload your `dropredo-tests/` folder. DropRedo creates the applications, environments, variables, groups and tests it declares.
2. Run tests from the app, on a schedule (hourly, daily or weekly), or from CI.

To run from CI, create an API token under **API Tokens**, copy a trigger ID from the **API** tab of an application's or group's Triggers page, and call:

```sh
curl -X POST "https://app.dropredo.com/api/v1/triggers/$DROPREDO_TRIGGER_ID/run" \
  -H "Authorization: Bearer $DROPREDO_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"browser": "chromium"}'
```

The response has the run's `execution_id` and a `report_url`: a public page with each test's steps, result and video, which you can post on a pull request.

To wait for the result, poll the run's status with the same token, every 15 seconds or so:

```sh
curl "https://app.dropredo.com/api/v1/executions/$DROPREDO_EXECUTION_ID" \
  -H "Authorization: Bearer $DROPREDO_API_TOKEN"
```

```json
{
  "success": {
    "execution_id": "…",
    "status": "failed",
    "finished": true,
    "passed": false,
    "counts": { "total": 12, "passed": 11, "failed": 1, "running": 0 },
    "report_url": "…",
    "app_url": "…"
  }
}
```

`status` is one of `running`, `waiting for manual action`, `completed`, `failed` or `force stopped`. Stop polling once `finished` is true, and fail your build unless `passed` is true. A run `waiting for manual action` is paused for a person and will not finish on its own. The **GitHub Action Workflow** download on the **API** tab does all of this for you.

## License

[MIT](LICENSE). Write, share and fork suites in this format freely.
