# Debug AI Notes: The Invisible Save

## Step 2: Claude's trace of the update method


The handle gets changed on the Python object in memory, but nothing ever
writes it to the database. The student.save() call is missing.

**What update does** (LearningAPI/views/student_view.py:77-100):

```python
84            student = NssUser.objects.get(pk=pk)          # SELECT row from DB -> Python object
...
87                if "slack_handle" in request.data:
88                    student.slack_handle = request.data["slack_handle"]   # changes the object in memory only
89                if "gitub_handle" in request.data:
90                    student.gitub_handle = request.data["gitub_handle"]
91                                                           # <- student.save() should be here
92                return Response(None, status=status.HTTP_204_NO_CONTENT)
```

**Why the change disappears:** In Django's ORM, a model instance is a Python
copy of a database row, not a live connection to it.

1. Line 84: `NssUser.objects.get(pk=pk)` runs a SELECT and builds a Python
   object from the row.
2. Line 88: `student.slack_handle = ...` changes an attribute on that object.
   No SQL runs.
3. Line 92: the view returns 204. The object is thrown away when the request
   ends, and the change goes with it.

Only `student.save()` sends the UPDATE statement to the database. The 204 is
misleading: it only means the code reached line 92 without an exception, not
that anything was saved.

**The fix:** Add the save between lines 90 and 92:

```python
                if "gitub_handle" in request.data:
                    student.gitub_handle = request.data["gitub_handle"]

                student.save()
                return Response(None, status=status.HTTP_204_NO_CONTENT)
```

**A second bug on lines 89-90:** The model field is spelled `github_handle`
(LearningAPI/models/people/nssuser.py:16), but the view checks for
`"gitub_handle"` (missing the "h") and sets `student.gitub_handle`. This means:
- If a client sends `github_handle` (the correct spelling), the check on
  line 89 is false, so the field is never touched.
- If a client sends `gitub_handle` (the typo), Python just adds a new
  attribute to the object that Django's ORM doesn't recognize as a real model
  field, so even after adding `.save()`, the GitHub handle still wouldn't
  persist.

Both lines should use `github_handle`.

**Takeaway (Claude's words):** "In Django, changing a field on a model
instance only changes it in memory. You need .save() to write it to the
database. When an 'update' returns success but nothing changes, a missing
.save() is one of the first things to check."

## Step 3: Comparison to debugger findings

Claude's explanation matches exactly what I observed with the debugger in the
manual exercise: execution reaches line 90, then jumps straight to the return
on line 92, with no save() call in between. Both approaches identified the
same root cause and the same missing line.

One thing Claude found that the debugger session did not: a second, separate
bug in the same method, a typo (gitub_handle instead of github_handle) that
means the GitHub handle field can never be updated, independent of the
missing .save() issue. The debugger trace only followed the slack_handle
path (per the assignment's breakpoint and step-through instructions), so this
second bug was outside what that exercise was looking for. The AI's static
read of the whole method surfaced it incidentally, since it wasn't
specifically asked to look for it, which is a genuine advantage of having it
read the full method rather than only stepping through one specific variable.

## Step 4: Claude's test

Claude pointed out the test file is already set up differently from how the
assignment describes: there's no setUp method, the user, NssUser, token,
client, and the PUT request are already written inline at the top of the
test, so most of "step 1" was already done.

Claude also corrected a detail: the existing PUT request uses
/students/{nss_user.id} with no trailing slash, not /students/{pk}/ as the
sample prompt suggested. It confirmed this by reading
LearningPlatform/urls.py:25, where the router is created with
DefaultRouter(trailing_slash=False), meaning a trailing slash would 404.

**The code to add** (replacing the final three lines: the two comments and
fail("Not Yet Implemented")):

```python
self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)

# Re-read the row from the database; the in-memory nss_user is stale
nss_user.refresh_from_db()
self.assertEqual(nss_user.slack_handle, "@newhandle")
```

Claude also noted the now-unused `from pytest import fail` import at the top
should be deleted.

**Why each line matters (Claude's explanation):**
- `assertEqual(response.status_code, 204)`: checks the view reported success.
  On its own this proves nothing, since the buggy code already returns 204.
- `nss_user.refresh_from_db()`: this is what makes the test useful. The
  nss_user object in the test is a separate Python object from the student
  object the view loaded internally; the view's changes never reach it.
  refresh_from_db() runs a fresh SELECT and reloads the fields from the
  database, so the test checks what was actually stored, not stale in-memory
  state.
- `assertEqual(nss_user.slack_handle, "@newhandle")`: the real check. Against
  the current buggy code, this fails with '@old_handle' != '@newhandle',
  correctly reproducing the bug. Once student.save() is added, it passes.

Claude's takeaway: "When you test a write, don't trust the response. Read the
data back from the database." Claude confirmed it did not edit the file or
run the test.

## Step 5: Test failure output

\`\`\`
LearningAPI/tests/test_student_update.py::StudentPartialUpdateTests::test_patch_slack_handle_persists_to_database FAILED

FAILURES
________ StudentPartialUpdateTests.test_patch_slack_handle_persists_to_database ________
LearningAPI/tests/test_student_update.py:40: in test_patch_slack_handle_persists_to_database
    self.assertEqual(nss_user.slack_handle, "@newhandle")
E   AssertionError: '@old_handle' != '@newhandle'
E   - @old_handle
E   + @newhandle

1 failed in 0.83s
\`\`\`

This confirms the test correctly detects the bug: the PUT request claimed
success, but refresh_from_db() shows the database still holds the original
'@old_handle' value, not the new one that was sent.

## Step 6: Claude's fix explanation

## Step 6: Claude's fix explanation

The line to add is: `student.save()`

**Where it goes:** on a new line between line 90 and line 92, indented to
match the return on line 92 (two levels inside the method body):

```python
86            if request.auth.user == student.user or request.auth.user.is_staff:
87                if "slack_handle" in request.data:
88                    student.slack_handle = request.data["slack_handle"]
89                if "gitub_handle" in request.data:
90                    student.gitub_handle = request.data["gitub_handle"]
91
                  student.save()          <- add here
92                return Response(None, status=status.HTTP_204_NO_CONTENT)
```

**Why that spot, specifically (Claude's reasoning):**
- After lines 87-90: all the field changes need to be made before saving, so
  a single UPDATE writes them together.
- Before line 92: once return runs, the method is finished, so any code after
  it never executes.
- Inside the if on line 86: only callers who pass the permission check should
  be able to save. At the same indent as line 86, unauthorized callers would
  get a save too.
- Not inside the `if "slack_handle"` block: at that indent, a request with
  only the GitHub field would never trigger a save.

Claude also reminded me that this only fixes the Slack handle; the GitHub
handle still won't persist until the gitub_handle typo on lines 89-90 is
corrected to github_handle (a separate bug it found in Step 2).

## Step 7: Test passing output

\`\`\`
LearningAPI/tests/test_student_update.py::StudentPartialUpdateTests::test_patch_slack_handle_persists_to_database PASSED

1 passed in 0.72s
\`\`\`

Applied the fix by typing `student.save()` myself (not pasted from Claude),
exactly at the location Claude described: after the slack_handle and
gitub_handle field assignments, before the return Response(...) line, and
inside the permission check's if block. The test went from failing to
passing after this single-line change.


Live verification: updated the Slack handle to "ai-fix-confirm-999" through
the Student interface, refreshed the page, and the new value persisted
correctly. This matches the test result; both the automated test and the
live UI confirm the fix works.