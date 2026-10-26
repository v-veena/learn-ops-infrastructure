# Conventions Notes

## JavaScript Linter (learn-ops-client)

### Configuration
Extends: "react-app" (from package.json eslintConfig), which itself extends an internal "./base" config inside the eslint-config-react-app package.
Plugins active: import, flowtype, jsx-a11y, react-hooks

### Rules I found (at least 5)
- no-unused-vars (warn): flags variables, imports, or parameters that are declared but never used.
- react-hooks/exhaustive-deps (warn): checks that useEffect/useCallback dependency arrays include every value the function actually references.
- react-hooks/rules-of-hooks (error): enforces that Hooks are only called at
  the top level of a component or custom hook, never conditionally or in loops.
- jsx-a11y/alt-text (warn): requires img elements to have an alt attribute for accessibility.
- react/jsx-no-target-blank (warn): flags target="_blank" links missing rel="noreferrer", a security risk since the new tab can otherwise access the opening page.
- eqeqeq (warn, "smart" mode): requires === instead of ==, with an exception allowed for comparing against null.
- no-throw-literal (warn): requires that anything thrown is a real Error object, not a plain string or literal value.

### First warning fixed
File and line: src/components/LearnOps.js, line 8
Rule: no-unused-vars
Fix: Removed the unused import `import { CohortProvider } from "./cohorts/CohortProvider"`,since CohortProvider was never referenced anywhere else in the file. Confirmed by rerunning `npx eslint src/`: the file no longer appears in the output, and the total problem count dropped from 285 to 284 (one fewer warning).

## Python Linter (learn-ops-api)

### Configuration
Disabled rules and what each would flag:
- broad-except: flags `except Exception:` (catching all exceptions), since catching everything can hide real bugs and make failures harder to  diagnose.
- arguments-differ: fires when a subclass overrides a method with a different  parameter signature than its parent. Disabled because Django ViewSets  override methods (like create, list, update) this way by design, so the  rule would constantly flag normal, correct Django code.
- missing-function-docstring: requires a docstring on every function. Disabled because descriptive test method names (like test_patch_slack_handle_persists_to_database) already communicate intent on their own, making a docstring redundant for many functions in this codebase.

Good names allowed: i, j, ex, pk
These are short variable names that would normally trigger the invalid-name rule (which expects longer, more descriptive names), but are allowed here because they're conventional and clear in context: i/j for loop counters,ex for exception variables in except blocks, and pk for Django's primary key convention (used constantly throughout the views, e.g. NssUser.objects.get(pk=pk)).

What pylint-django likely adds: pylint doesn't understand Django's dynamic ORM magic by default, so without pylint-django, it reports false positives like "Class 'NssUser' has no 'objects' member", since .objects is added dynamically by Django's Model base class, not written explicitly in the class body. pylint-django teaches pylint to recognize Django-specific patterns (model managers, querysets, serializers, viewsets) so these aren't flagged as errors. (The pylint output in this run shows many E1101 "no-member"
warnings for .objects and .DoesNotExist, which is exactly the kind of noise pylint-django is meant to suppress; it may not be installed/active here, which is why they still appear.)

### First warning fixed
File and line: LearningAPI/admin.py, line 1 
Rule and category: missing-module-docstring (C0114, Convention)
Fix: Added a one-line module docstring, `"""Admin site configuration for the LearningAPI application."""`, as the very first line of the file, before any imports. Confirmed by rerunning pylint: the C0114 message for admin.py line 1 no longer appears in the output