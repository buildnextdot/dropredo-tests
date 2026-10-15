# Generate a DropRedo test-suite import file

You are working inside the source repository of a web application. Your job is to
read this codebase and write a **`dropredo-tests/` folder** describing an end-to-end
browser test suite for it, in DropRedo's import format.

DropRedo will read these files, create the applications/environments/variables/groups
they declare, and then hand each test's plain-English steps to an AI agent that drives
a real browser to synthesise Playwright actions and assertions. **You are writing
intent, not code.** Never write CSS selectors, XPath, or Playwright calls.

---

## Output contract

Create a folder `dropredo-tests/` in the repository root, containing:

```
dropredo-tests/
  dropredo.yaml            # manifest — applications, environments, variables. NO groups.
  <module>.yaml            # one file per feature area, each declaring one group
  <area>/<module>.yaml     # nest when a product area has several modules
```

- **One file per feature area**, named after the group it declares:
  `authentication.yaml`, `checkout.yaml`, `admin/billing.yaml`. Keep each file under
  ~300 lines; if a group outgrows that, split the group itself.
- Every file uses the **same schema** and may declare any subset of the tree. A module
  file repeats only the application name, then its own `groups:` — it never repeats
  environments or variables.
- DropRedo merges all the files before validating, so a test in `checkout.yaml` may
  `depends_on` a test declared in `authentication.yaml`. Write dependencies freely
  across files.
- Do not modify any other file in the repository, and do not add an index, a
  `README`, or any `includes:` directive — there is no such thing; splitting is
  purely by file.

After writing, print: the file tree you created, the number of
applications / environments / groups / tests, a list of every `REPLACE_ME` value the
user must fill in, any part of the app you deliberately did not cover and why, and
every test that creates data with no cleanup (and why — see "Clean up what you
create" below).

### Rules that span files

- **Declare each environment and variable exactly once**, in `dropredo.yaml`. Declaring
  the same variable key twice with different values is a hard error.
- **Declare each test exactly once.** The same test name in the same group in two files
  is a hard error.
- **Never split one group across two files.** One group, one file.

---

## Phase 1 — Recon (read-only)

Build a picture of the app before writing anything. Look for, in roughly this order:

1. **Existing e2e tests** — `cypress/`, `e2e/`, `tests/e2e/`, `playwright.config.*`,
   `*.spec.ts`, `*.cy.js`. This is the richest source: it already encodes the critical
   flows, the selectors that work, and the test data. Mine it before anything else.
2. **Routes** — `routes/web.php` (Laravel), `app/` or `pages/` (Next.js), `src/App.tsx`
   or router config (React/Vue SPA), `config/routes.rb` (Rails), `urls.py` (Django).
   Public vs authenticated routes tell you the group structure.
3. **Auth** — middleware, guards, `AuthProvider`, login/register/logout controllers and
   components. Almost every suite needs a login test that everything else depends on.
4. **Environment config** — `.env.example`, `config/`, `docker-compose.yml`,
   `vercel.json`. Extract base URLs and the *names* of environments the team actually
   uses (`prod`, `staging`, `dev`, `qa`).
5. **Forms and key components** — read the JSX/Blade/Vue for the flows you plan to test
   and note the **visible labels and button text**. Those exact strings are what make a
   step unambiguous to the browser agent.
6. **Seeders / factories / fixtures** — often contain the test account credentials.
7. **Delete / cancel / deactivate / remove features** — for every create flow you plan
   to test (delete post, cancel order, deactivate account, remove team member). You
   will need these to clean up after each test — see "Clean up what you create" below.
   Note any create flow that has no matching one.
8. **TOTP / 2FA / CAPTCHA / manual-approval / off-app sign-in steps** — any point in a
   flow where a human has to do something the browser agent cannot (enter a rotating
   code, solve a CAPTCHA, approve on another device, complete a real OAuth login on
   another site). Flag the step whose action reaches that point — the click that opens
   it, the field that needs it — with `pause_for_manual_action: true` (see Field
   reference). The run executes that step as normal, then pauses immediately
   afterward and waits for a human instead of failing. Do not flag the step that
   comes after the manual part; flag the one that leads into it.

Do not run the app, install dependencies, or execute tests.

## Phase 2 — Decide the structure

- **applications** — usually one. Split only if the repo genuinely hosts separate
  products (e.g. a customer storefront and an internal admin panel on different domains).
- **environments** — one per deployment target the team really has. If you can only find
  one base URL, declare one environment. Do not invent a `staging` that doesn't exist.
- **variables** — every URL, email, password, API key or account id that differs between
  environments or is a secret. Never inline a credential into a step.
- **groups** — by user-facing feature area, matching how the app's own code is organised:
  `authentication`, `checkout`, `user-profile`, `admin-billing`. Aim for 3–8 groups.
- **tests** — one user-visible outcome each. Prefer 5–15 steps per test. If a flow is
  longer, split it and chain the parts with `depends_on`.
- **tests that create data** — pair every create flow with cleanup, either more steps
  at the end of the same test or a dependent test that removes what was created. See
  "Clean up what you create" below before you write any test that signs up, creates,
  posts, uploads, invites, subscribes, or orders something.

Cover the critical paths first: sign-up, sign-in, the app's single most important
workflow, and sign-out. Breadth over depth — a shallow test of every major feature is
more useful than an exhaustive test of one.

## Phase 3 — Write the files

Write `dropredo-tests/dropredo.yaml` first (applications, environments, variables), then
one module file per group.

---

## Schema

Every file — manifest and module alike — uses this shape and declares whatever subset of
it applies:

```yaml
version: 1

applications:
  <application name>:
    description: <optional>
    environments:
      <environment name>:              # FIRST key listed is where generation runs
        description: <optional>
        variables:
          <key>: "<value>"             # scalar shorthand, type defaults to string
          <key>:
            value: "<value>"
            type: number               # string | number | boolean | file | null
    groups:
      <group name>:
        description: <optional>
        tests:                         # this wrapper key is REQUIRED
          <test name>:
            description: <optional>
            environments: [<env>, ...] # ordered; [0] is the generation environment
            depends_on: []             # 0 or 1 test names
            tabs:                      # optional — 0–4 named browser tabs; the first
              - <tab name>              #   is where a step with no `tab:` runs
            use-parent-test-session-in-tabs:  # optional — tab names that resume
              - <tab name>                    #   depends_on's signed-in session
            steps:
              - <plain english step>                # no assertions
              - step: <plain english step>
                pause_for_manual_action: true      # step needs a human: TOTP, CAPTCHA, approval
                tab: <tab name>                     # which declared tab this step runs in
                assertions:
                  - <plain english assertion>       # terse
                  - description: <plain english assertion>
                    negate: true                    # assert the opposite
                    type: <assertion type>          # optional pin
                    expected: <value>               # optional pin
                    attribute: <attr name>          # required for elementHasAttribute / elementHasCSS
```

### Field reference

| Field | Required | Notes |
|---|---|---|
| `version` | no | must be `1` if present |
| `environments` (on a test) | **yes** | every name must be declared under the same application |
| `depends_on` | no | at most **one** entry — see Hard Rules |
| `steps` | **yes** | non-empty |
| `step` / `title` | yes (map form) | the plain-English instruction |

| `action` | no | pin only if the phrasing is ambiguous. One of: goto, reload, fill, type, press, click, hover, check, uncheck, select, wait, waitUntil, dblclick, rightClick, clear, focus, blur, setChecked, selectText, scrollIntoView, scrollPage, goBack, goForward, waitForURL, waitForLoadState, dragAndDrop, uploadFile, downloadFile |
| `tabs` (on a test) | no | at most 4 names; the first is the primary tab — a step with no `tab:` runs there |

| `use-parent-test-session-in-tabs` | no | tab names that resume the `depends_on` parent's signed-in session; requires a `depends_on` |
| `tab` (on a step) | no | one of the test's declared `tabs`; omit for the primary tab |
| `pause_for_manual_action` | no | `true` pauses the run before this step and waits for a human — a TOTP, a CAPTCHA, an approval on another device |
| `assertions[].description` | yes (map form) | the plain-English fact being asserted |

| `assertions[].type` | no | one of: status, checkboxIsChecked, elementIsDisabled, elementIsEnabled, elementIsFocused, elementContainsText, listHasExactNumberOfChildren, elementMatchesText, inputHasValue, selectHasOptionsSelected, pageHasTitle, pageHasURL, elementIsVisible, elementIsHidden, elementIsAttached, elementIsEditable, elementIsEmpty, elementInViewport, elementHasAttribute, elementHasClass, elementHasId, elementHasCSS, elementHasRole, elementHasAccessibleName, elementHasAccessibleDescription, responseIsOk, downloadStarted |

| `assertions[].negate` | no | `true` asserts the fact is **not** true |
| `assertions[].expected` | no | pin only if the phrasing is ambiguous — the exact value the generator should check for |
| `assertions[].attribute` | conditional | required when `type` is `elementHasAttribute` or `elementHasCSS` |

Every pinned field above is used exactly as written — the generator never
second-guesses a value you supplied, it only resolves the ones you left out.
Never write `selector:` — there is no such field here; the whole point of this
format is that you describe *what* is true, and the generator finds the
element. A pinned `type` that needs an `attribute` (see above) without one is
a hard validation error, not a silent guess.

---

## Hard rules — violating any of these fails validation on upload

1. **Every variable must have a non-empty value.** `base_uri:` with nothing after it is
   rejected. If you cannot determine a real value, write `"REPLACE_ME"` and list it in
   your summary. Never leave it blank.
2. **Variable keys match `^[a-zA-Z0-9_-]+$`.** No dots, spaces or `@`. `base_uri` ✓,
   `base.uri` ✗.
3. **`depends_on` holds at most one test name.** A test inherits the browser session of
   exactly one parent. To express "needs A and B", chain them: `B depends_on A`,
   `C depends_on B`.
4. **The dependency target must be declared in the same application, and must run in every
   environment its dependant runs in.** If `logout` targets `[prod, staging]` then its
   parent `login` must target at least `[prod, staging]`.
5. **Every step's `tab:` must name a tab the same test declares under `tabs:`.** A step
   with no `tab:` runs in the first declared tab (the primary).
6. **`use-parent-test-session-in-tabs` requires a `depends_on`, and every name in it
   must appear in `tabs:`.** Without a parent there is no session to resume.
7. **A tab name must be unique within its test and must not contain a `{{ token }}`.**

8. **No dependency cycles**, and no chain deeper than 20.

9. **Every `{{ token }}` you use must be declared as a variable in every environment that
   test runs in.** A `{{ base_uri }}` used by a test targeting `[prod, staging]` must exist
   under both.
10. **Tests must sit under a `tests:` key**, not directly under the group name.
11. **Names are unique within their scope** and at most 255 characters.

12. Limits: at most 10 applications,
   500 test-environment pairs,
   100 steps per test,
   20 assertions per step,
   4 tabs per test.


---

## Writing good steps

A browser agent reads each step, looks at the live page, and decides what to do. Write
what a careful human tester would say out loud.

**One action per step.**
- ✅ `fill the {{ admin_user_email }} into the email field`
- ❌ `fill in the login form and submit it`

**Name the thing the way the user sees it.** Quote exact visible text from the codebase.
- ✅ `click the button with text "Add to cart"`
- ❌ `click the primary CTA`
- ❌ `click button.btn-primary` ← never write selectors

**Start every root test by navigating.**
- ✅ `visit {{ base_uri }}/login`

**Put every credential, URL and account id in a variable.**
- ✅ `fill the {{ admin_user_password }} into the password field`
- ❌ `fill "hunter2" into the password field`

**Disambiguate when a page has repeats.**
- ✅ `click the "Delete" button in the first row of the orders table`
- ❌ `click Delete`

**Don't test the backend.** No API calls, no DB assertions, no email inbox checks — this
is a browser agent.

**Waiting is usually unnecessary** — the agent settles the page between steps. Add an
explicit wait step only for something genuinely asynchronous, and say why.

## Multi-tab tests

A test may declare up to a few named browser tabs and assign each step to one with
`tab:`. Steps still run in **one flat ordered list**, regardless of tab — the agent just
switches which page it acts on. Each tab is its own signed-in-or-not browser session: a
tab in `use-parent-test-session-in-tabs` resumes the `depends_on` parent's session, every
other tab starts fresh, and signing in on one tab does **not** sign in another. A child
test that later `depends_on` this one inherits only the **primary** tab's session (the
first name under `tabs:`).

```yaml
tabs:
  - admin panel
  - blog
use-parent-test-session-in-tabs:
  - admin panel
steps:
  - step: visit {{ base_uri }}/admin
    tab: admin panel
    assertions:
      - the heading "Admin Panel" is visible
  - step: click the button with text "Publish"
    tab: admin panel
  - step: visit {{ base_uri }}/blog
    tab: blog
    assertions:
      - the heading "The blog" is visible
```

**Only reach for tabs when a flow genuinely needs two pages open at once** — an admin
change verified on the public site, a two-account chat, an action in one tab confirmed by
an email or notification tab. **A flow that simply visits two pages in sequence needs no
tabs at all** — that is just two ordinary steps in one tab. Adding tabs to a test that
doesn't need simultaneity is the most common mistake here; when in doubt, leave `tabs:`
out entirely.

## Writing good assertions

Assertions hang off the step **after which the fact becomes true**.

**One fact per assertion.**
- ✅ `the heading "Dashboard" is visible` + `the URL contains /dashboard` (two assertions)
- ❌ `the dashboard loads correctly with the right URL`

**Assert what a user can see**, in the same visible-text vocabulary as the steps.

**Quote text that appears inside ONE element.** The generator finds the element that
carries the text you quote. Naming a container and quoting text from somewhere inside it
makes those two things disagree, and the assertion checks the wrong element.
- ✅ `the text "Authoring model" is visible`
- ❌ `the comparison table shows the row "Authoring model"` ← the table is not the row

**Never pair a container with a value from inside it — split it into two assertions.**
- ❌ `the "Starter" plan card shows a price of "₹1,299"`
- ✅ `the text "Starter" is visible` **+** `the text "₹1,299" is visible`

**Every assertion must name its target with text from the page.** A description the
generator cannot ground in something visible produces no selector at all.
- ❌ `an answer is visible below the "What is DropRedo?" question`
- ✅ `the text "DropRedo runs real browser tests" is visible`

**For anything with no visible text** — a video, a canvas, a chart, an icon-only button —
name it by test id or aria-label, or assert on visible text beside it instead. An element
with no text and no label is not reliably findable, and the assertion will fail even
though the page is fine.
- ❌ `a video player with controls is visible`
- ✅ `the video with test id "demo-player" is visible`
- ✅ `the text "Watch the 90-second demo" is visible` ← assert the caption instead

**For a link's destination, pin all three fields.** Left to plain English this is the one
assertion shape that reliably goes wrong, because "links to" does not say *which*
attribute to read:
```yaml
- description: the "Subscribe" button links to the app
  type: elementHasAttribute
  attribute: href
  expected: https://app.dropredo.com
```
`expected` is matched **exactly**, so write the full attribute value — including any
`mailto:` prefix and query string (`mailto:sales@acme.com?subject=Enterprise`). To check
only that the link exists, leave `expected` out entirely.

**Quote formatted values exactly as rendered**, including currency symbols and thousands
separators: `"₹1,299"`, not `1299`.

**Content behind a click must be asserted after that click.** Anything inside a collapsed
accordion, `<details>` or closed dropdown counts as hidden and fails a visibility
assertion, so the step that opens it has to come first.

**Use `negate: true` for absence** rather than phrasing it negatively — the generator
handles absence specially and it is more reliable:
```yaml
- description: an error message is shown
  negate: true
```

**Pin `type` / `expected` only when plain English is ambiguous.** Prefer leaving them out;
the generator resolves them from the live page and gets it right more often than a guess.
When you do pin `pageHasURL`, give the full expected URL. The exceptions are the two
shapes above — link destinations, and anything without visible text — where pinning is
the reliable option rather than the fallback.

**Assert something after every meaningful step** — a step with no assertion proves only
that the action did not throw. At minimum, assert the outcome of every navigation and
every form submission.

---

## Clean up what you create

Nothing in DropRedo's schema enforces this — it is entirely on you to design tests this
way. Every time the suite runs, a test that creates data (signs up a user, places an
order, posts a comment, uploads a file, invites a team member) leaves that record
behind unless something removes it. Run the suite daily with no cleanup and the
environment silently fills with thousands of throwaway users and orders. Design every
test that creates something as if it must leave the environment exactly as it found it.

**Rule: every test that creates a persistent record must also remove it, through the
app's own UI.** You cannot call an API or touch the database directly — this is still a
browser agent driving the real UI, the same as every other step. Two ways to do it:

1. **Same test, at the end — the default.** Once you've asserted the create succeeded,
   add the steps to delete/cancel/deactivate that same record, using whatever the app's
   own UI offers (a "Delete account" button, a "Cancel order" action, a "Remove" icon on
   a list row). Prefer this whenever the feature exists.

   ```yaml
   create and delete a support ticket:
     environments: [prod]
     steps:
       - visit {{ base_uri }}/tickets/new
       - fill the "Order is missing" text into the subject field
       - step: click the button with text "Submit ticket"
         assertions:
           - the heading "Ticket submitted" is visible
       - step: click the button with text "Delete ticket"
         assertions:
           - the ticket "Order is missing" is not in the tickets list
   ```

2. **A dependent cleanup test**, when the create and its natural delete live on
   different pages or flows (e.g. sign-up is a public form, but deleting the account
   happens from account settings). Write the deletion as its own test that
   `depends_on` the test that created the record — it resumes the exact browser session
   the create test finished with, already signed in as the account it is about to
   delete.

   ```yaml
   user can sign up:
     environments: [prod]
     steps:
       - visit {{ base_uri }}/signup
       - fill the {{ new_user_email }} into the email field
       - step: click the button with text "Create account"
         assertions:
           - the heading "Welcome" is visible

   user can delete the account they just created:
     environments: [prod]
     depends_on: [user can sign up]
     steps:
       - visit {{ base_uri }}/account/settings
       - step: click the button with text "Delete my account"
         assertions:
           - the heading "Sign in" is visible
   ```

**If a create flow has no matching delete/cancel/deactivate feature in the app, you
cannot fabricate one.** Say so plainly in your final summary (e.g. "`user can place an
order` has no cleanup: the app has no way to cancel or delete an order") instead of
silently leaving it unclean.

**Prefer reusing one fixed record over creating a new one every run**, wherever the UI
allows it. A test that edits the same "Test Ticket #1" each time needs no cleanup at
all; a test that always creates a brand-new record needs a delete step every time.
Check whether the flow has a natural edit-if-exists shape before assuming you must
create-then-delete.

**A failed assertion aborts the rest of the test, including any cleanup after it** —
this is a real limitation of running steps in sequence, not something you can work
around in YAML. Keep the create → assert → cleanup chain short and assert only what
you must in between, so a genuine bug is what skips a cleanup step, not an unrelated
flaky assertion earlier in the test.

**This does not apply to read-only or idempotent actions** — signing in, viewing a
page, searching, filtering. Only pair cleanup with actions that write a new row: sign
up, create, post, upload, invite, subscribe, place an order.

---

## Worked example

Given this in the repo:

```php
// routes/web.php
Route::get('/login', [AuthController::class, 'show'])->name('login');
Route::post('/login', [AuthController::class, 'store']);
Route::middleware('auth')->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
    Route::post('/logout', [AuthController::class, 'destroy']);
});
```
```blade
{{-- resources/views/auth/login.blade.php --}}
<h1>Sign in to Acme</h1>
<input type="email" name="email" placeholder="Email address">
<input type="password" name="password" placeholder="Password">
<button type="submit">Sign in</button>
```
```
# .env.example
APP_URL=https://acme.test
```

You would write two files.

**`dropredo-tests/dropredo.yaml`** — the manifest. No groups.

```yaml
version: 1

applications:
  acme:
    description: Acme customer web app
    environments:
      prod:
        variables:
          base_uri: "https://acme.test"
          user_email: "REPLACE_ME"
          user_password: "REPLACE_ME"
```

**`dropredo-tests/authentication.yaml`** — repeats only the application name.

```yaml
applications:
  acme:
    groups:
      authentication:
        description: Sign in and sign out
        tests:
          user can sign in with valid credentials:
            environments: [prod]
            depends_on: []
            steps:
              - step: visit {{ base_uri }}/login
                assertions:
                  - the heading "Sign in to Acme" is visible
              - fill the {{ user_email }} into the email address field
              - fill the {{ user_password }} into the password field
              - step: click the button with text "Sign in"
                assertions:
                  - the browser is on the dashboard page
                  - description: an invalid credentials error is shown
                    negate: true

          user can sign out:
            environments: [prod]
            depends_on:
              - user can sign in with valid credentials
            steps:
              - step: click the button with text "Log out"
                assertions:
                  - the heading "Sign in to Acme" is visible
```

Note what happened: the visible strings `Sign in to Acme` and `Sign in` came straight out
of the template; the credentials became `REPLACE_ME` variables rather than invented
values; `sign out` did not repeat the login steps, it declared `depends_on` instead; and
the module file repeated the application name but **not** the environments or variables.

A third file, `dropredo-tests/checkout.yaml`, would follow the same shape and could
declare `depends_on: [user can sign in with valid credentials]` even though that test
lives in `authentication.yaml` — the files are merged before anything is resolved.

---

## Before you finish — self-check

Re-read every file you wrote and confirm each line:

- [ ] Everything lives under `dropredo-tests/`; nothing else in the repo was touched.
- [ ] `dropredo.yaml` declares applications/environments/variables and **no** groups.
- [ ] Each module file declares exactly one group and repeats only the application name.
- [ ] No environment or variable is declared in more than one file.
- [ ] No test name is declared in more than one file.
- [ ] Every variable has a non-empty value; every unknown one is exactly `REPLACE_ME`.
- [ ] No credential, URL or account id appears literally inside a step.
- [ ] Every `{{ token }}` is declared in **every** environment its test targets.
- [ ] Every `depends_on` has 0 or 1 entries, names a test in the same application, and
      that parent covers all of the child's environments.

- [ ] No cycles; no chain longer than 20.

- [ ] Every step's `tab:` names a tab declared under that same test's `tabs:`.
- [ ] Every name in `use-parent-test-session-in-tabs` is declared under `tabs:`, and the
      test has a `depends_on` if it uses this field at all.
- [ ] Tab names are unique within their test and contain no `{{ token }}`.
- [ ] Every group uses the `tests:` wrapper key.
- [ ] No CSS selectors, XPath, or code in any step or assertion.
- [ ] Every step is a single action; every assertion is a single fact.
- [ ] Every assertion quotes text that appears inside a single element, and never pairs a
      container with a value from inside it.
- [ ] Every assertion names its target with text, a label, or a test id — nothing vague
      like "an answer" or "a video player".
- [ ] Every link-destination assertion pins `type: elementHasAttribute`, `attribute: href`
      and the exact full `expected` value.
- [ ] Nothing behind a click (accordion, `<details>`, dropdown) is asserted before the step
      that opens it.
- [ ] Every navigation and every form submission has at least one assertion after it.
- [ ] Every test that creates a record deletes/cancels/deactivates it by the end of the
      same test or a dependent cleanup test, through the app's own UI — or, if no such
      feature exists, that is named in your summary.
- [ ] Where the app supports editing an existing record instead of creating a new one,
      you reused it rather than creating a fresh record every run.
- [ ] Any `action` / `type` you pinned is in the allowed lists above, spelled exactly.
- [ ] `attribute` is present wherever `type` is `elementHasAttribute` or `elementHasCSS`.
- [ ] Every file is valid YAML (2-space indent, quote any value containing `:` or `{{`).

Then print the file tree, your summary and the `REPLACE_ME` list.

