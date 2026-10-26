# Conventions Notes (AI)

## Step 3: /conventions learn-ops-client

Summary: The client uses one linter: ESLint. No Python config files exist
here (.pylintrc, setup.cfg, pyproject.toml), since this is a JavaScript/React
app. No standalone .eslintrc*, Prettier, or Stylelint configs either.

Tool: ESLint (v8.41.0, installed with react-scripts 5)

How to run it:
cd ~/workspace/lms/learn-ops-client && npx eslint src --ext .js,.jsx
- Auto-fix: add --fix
- One file: npx eslint src/components/utils/HumanDate.js
- Also runs automatically inside npm start and npm run build

Where configured: package.json, eslintConfig key -> "extends": "react-app".
No project-specific rule overrides; everything comes from
node_modules/eslint-config-react-app/index.js (plus base.js and jest.js).

Rule categories with examples:
- Core JS correctness (eslint core): no-undef (error), no-unused-expressions
  (error), no-dupe-keys, no-unreachable, no-const-assign, use-isnan,
  valid-typeof (warn)
- Code hygiene (eslint core): no-unused-vars (warn), no-useless-escape,
  no-mixed-operators, no-throw-literal, default-case, array-callback-return
  (warn)
- Security-ish (eslint core): no-eval, no-implied-eval, no-new-func,
  no-script-url (warn), no-restricted-globals (error)
- React (eslint-plugin-react): react/jsx-no-undef (error), react/no-typos
  (error), react/require-render-return (error), react/jsx-no-target-blank,
  react/jsx-no-duplicate-props, react/no-direct-mutation-state (warn)
- Hooks (eslint-plugin-react-hooks): react-hooks/rules-of-hooks (error),
  react-hooks/exhaustive-deps (warn)
- Accessibility (eslint-plugin-jsx-a11y): jsx-a11y/alt-text,
  jsx-a11y/anchor-has-content, jsx-a11y/aria-props,
  jsx-a11y/heading-has-content (warn)
- Imports (eslint-plugin-import): import/first, import/no-amd,
  import/no-webpack-loader-syntax (error), import/no-anonymous-default-export
  (warn)
- Flow/TS: only applies to .ts/.tsx or Flow-annotated files, none exist here
- Tests (eslint-plugin-jest, testing-library): applies to *.test.js files

Rules turned off on purpose:
1. In the preset (TypeScript override only): for .ts/.tsx files the preset
   disables core rules like no-undef, no-unused-vars in favor of
   @typescript-eslint/* equivalents. No effect here since there are no
   TypeScript files.
2. In the source code: 4 eslint-disable comments, found by re-running with
   --no-inline-config:
   - HumanDate.js:21, hides no-useless-escape on an unnecessary / escape
   - StudentDashboard.js:20, likely hides no-unused-vars on firstName
     (not fully confirmed)
   - StudentCardList.js:152 and StudentCapstoneList.js:55, both
     /* eslint-disable no-undef */ blocks meant to allow structuredClone,
     but ESLint 8.41 already recognizes that global, so the comments do
     nothing today. Also flagged a real risk: these are BLOCK-style
     disables with no matching eslint-enable, so no-undef stays off from
     that line to the end of the file, not just one line.
3. Editor setting: .vscode/settings.json sets
   "javascript.validate.enable": false, which disables VS Code's built-in
   TypeScript-based JS checking (not ESLint itself).

Current state: 284 problems (4 errors, 280 warnings). Breakdown:
no-unused-vars (188), react-hooks/exhaustive-deps (60),
react/jsx-no-target-blank (12), array-callback-return (6), jsx-a11y/alt-text
and no-undef (4 each), other (10).

All 4 errors are no-undef in MiniCoreSkillSliders.js (note, setNote,
toggleSkillHistory used but never defined), flagged as likely real bugs that
would throw ReferenceError at runtime.

CI note: .github/workflows/main.yml only runs npm install and
npm run build:production, with CI=false, so warnings never fail the build.
No separate lint step exists in CI.

## Step 4: /conventions learn-ops-api

Summary: The API uses one linter: Pylint, configured in .pylintrc. No ESLint,
setup.cfg, or pyproject.toml. autopep8 is in the Pipfile but unconfigured and
unused by any script.

Tool: Pylint (v4.0.7, Python 3.11.11)

How to run it: Pylint is not installed on the host machine, only inside the
Docker image (Dockerfile runs pipenv install --system --dev).
  docker exec learning-platform-api sh -c 'cd /app && pylint LearningAPI LearningPlatform LogViewer'
One file: docker exec learning-platform-api pylint LearningAPI/views/student_view.py
(Uses docker exec with the container name directly, rather than
docker compose exec api; functionally similar, but exec targets the
container name and works from any directory, while compose exec needs the
service name and the directory with docker-compose.yml.)

Where configured: ~/workspace/lms/learn-ops-api/.pylintrc (6 lines total):
[FORMAT] good-names=i,j,ex,pk
[MESSAGES CONTROL] disable=broad-except,arguments-differ,missing-function-docstring
Everything else uses Pylint defaults (e.g. 100-char max line length).

Rule categories with examples (counts from this actual codebase run):
- E Error (probable bugs): no-member E1101 (335), invalid-str-returned E0307
- W Warning: unused-argument (38), unused-import (24), unnecessary-pass (11),
  bad-indentation (7), fixme (9)
- C Convention (style/PEP8): line-too-long (273), missing-class-docstring
  (182), missing-module-docstring (167), invalid-name (89),
  wrong-import-order (19)
- R Refactor (design smells): too-few-public-methods (72), no-else-return
  (31), duplicate-code (31), too-many-ancestors (14), cyclic-import (7)

Current result: 1,436 messages, score 3.08/10.

good-names explained: i,j,ex,pk replaces Pylint's default list (i,j,k,ex,
Run,_), so k and _ are no longer allowed by default; pk was added
specifically because Django/DRF use it everywhere (e.g.
def retrieve(self, request, pk=None)).

Disabled rules, explained with real hit counts if re-enabled:
- broad-except (W0718 / broad-exception-caught): flags
  except Exception as ex: blocks that catch everything instead of specific
  exceptions. These views use that pattern on purpose to return 400/500
  responses. 75 hits if re-enabled.
- arguments-differ (W0221): flags a method overriding a parent with a
  different signature, e.g. StudentViewSet.retrieve(self, request, pk=None)
  dropping the parent's *args/**kwargs. DRF viewsets do this intentionally.
  18 hits if re-enabled.
- missing-function-docstring (C0116): flags any function/method without a
  docstring. 90 hits if re-enabled.

Inline # pylint: disable comments found in the code (8 total), with specific
file:line locations and reasons, e.g. invalid-str-returned on __str__ methods
(Pylint can't prove a model field value is a str) and no-member on Django ORM
relationship access patterns Pylint doesn't understand statically.

Extra findings beyond what was asked:
1. pylint-django is installed (in the Pipfile) but NOT activated (.pylintrc
   has no load-plugins=pylint_django line). The skill actually ran Pylint
   WITH the plugin enabled to test this:
     docker exec learning-platform-api sh -c 'cd /app && pylint LearningAPI --load-plugins=pylint_django --django-settings-module=LearningPlatform.settings'
   Score jumped from ~3 to 7.54/10, confirming the 335 no-member errors are
   almost all false positives caused by Pylint not understanding Django's
   dynamic ORM attributes. Enabling the plugin would be a one-line .pylintrc
   change (not made, since the skill only reports).
2. CI doesn't run Pylint at all; .github/workflows/main.yml only installs
   dependencies and restarts the service, so lint issues never block a
   merge. My own earlier manual fix (commit 7e92bf2, adding the
   missing-module-docstring to admin.py) is exactly the kind of manual-only
   fix this gap requires.
3. Many remaining messages (too-few-public-methods on Meta/serializer
   classes, too-many-ancestors on DRF ViewSets) are expected, normal patterns
   for Django code that Pylint doesn't recognize without pylint-django.

## Step 5: Refinement

Refined the prompt to add: "If the tool only runs inside a container (not
installed on the host), check whether a docker-compose.yml exists at or
above $ARGUMENTS. If one exists, prefer the command form
`docker compose exec <service-name> <command>` ... over a bare
`docker exec <container-name> <command>`... State clearly which directory the
command must be run from."

Result after rerunning against learn-ops-api:
- Before: `docker exec learning-platform-api sh -c 'cd /app && pylint ...'`
  (worked, but didn't match the project's documented convention and gave no
  directory to run from)
- After: `cd ~/workspace/lms/learn-ops-infrastructure` then
  `docker compose exec api pylint LearningAPI LearningPlatform LogViewer`,
  explicitly matching the command style used throughout the manual exercise,
  with the required working directory stated clearly.

The second run was also more rigorous generally: it measured exact hit
counts for each disabled rule by running `--disable=all --enable=<rule>`
individually (broad-except: 75, arguments-differ: 18,
missing-function-docstring: 90), and it caught that broad-except was renamed
to broad-exception-caught in newer Pylint versions, meaning a naive search
for the old rule name in fresh pylint output would find nothing even though
the rule is still active under its new name.

## Step 6: Comparison

**What matched my manual notes:**
- Both identified ESLint (via "extends": "react-app" in package.json) for
  the client, and Pylint (via .pylintrc) for the API.
- Both correctly listed the three disabled Pylint rules
  (broad-except, arguments-differ, missing-function-docstring) and the
  good-names setting (i,j,ex,pk).
- Both named well over 5 specific ESLint rules across multiple plugin
  categories.

**What the skill got wrong or missed on the first run:**
- The pylint run command used `docker exec <container-name>` instead of this
  project's documented `docker compose exec <service-name>` convention, and
  didn't state which directory to run the command from. My manual notes
  used `docker compose exec api pylint LearningAPI/` run from
  learn-ops-infrastructure, since that's the pattern established throughout
  the course exercises.

**What changed after refining the prompt:**
- Added an explicit instruction to detect docker-compose.yml and prefer the
  compose-based command form, with the working directory stated. The second
  run correctly produced `docker compose exec api pylint ...`, run from
  learn-ops-infrastructure, matching my manual notes exactly.

**What the skill found that I missed manually:**
- Exact version numbers (ESLint 8.41.0, Pylint 4.0.7, astroid 4.0.4).
- Actual hit counts for every disabled rule, obtained by actually running
  Pylint with each rule individually re-enabled, rather than just describing
  the rule in the abstract.
- The pylint-django plugin is installed but never activated; the skill
  proved this by actually re-running Pylint WITH the plugin loaded and
  showing the score jump from 3.08/10 to 7.55/10 (confirming most "no-member"
  errors are false positives from Django's dynamic ORM attributes).
- Four inline eslint-disable comments in the client code and what each one
  actually suppresses, including a real bug: two of them are block-style
  disables with no matching re-enable, so they silence no-undef for the rest
  of the file, not just one line.
- Four real no-undef errors in MiniCoreSkillSliders.js that look like actual
  runtime bugs (undefined variables referenced), not just style nitpicks.
- That broad-except was renamed to broad-exception-caught in newer Pylint,
  so searching pylint output for the old name would miss it.
- Neither repo enforces linting in CI; both pipelines only install
  dependencies and build/restart, so all of this relies on people running
  the linters manually.