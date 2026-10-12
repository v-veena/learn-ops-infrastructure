# Debug Notes: The Invisible Save

## 1. Bug report summary

Updating the Slack handle on the student profile page appeared to work: the
form submitted with no error, and the UI reflected the new value immediately.
However, refreshing the page showed the old value again, with no error
anywhere in the Network tab. Reproducing it confirmed this: a PUT request to
/students/{id} returned 204 No Content (success), but a subsequent GET request
for the profile still returned the original slack_handle, unchanged.

## 2. What the debugger showed

| Question | Answer |
|----------|--------|
| Value of `student.slack_handle` before line 87 | The original value already in the database (the old handle) |
| Value of `student.slack_handle` after line 88 | The new value sent in the request body |
| Next line executed after line 90 | Line 92, `return Response(None, status=status.HTTP_204_NO_CONTENT)` — execution jumped straight to the return with nothing in between |

## 3. Root cause

The `update` method set `student.slack_handle` and `student.github_handle`
directly on the Python model instance, but never called `student.save()`.
In Django's ORM, assigning to a model instance's attribute only changes the
in-memory Python object; it does not touch the database. Only an explicit
`.save()` call issues the SQL UPDATE statement. Without it, the view still
returned a 204 success response (because no exception was raised), making the
failure invisible: the API claimed success while the change was silently
discarded.

## 4. The fix

Added a single line, `student.save()`, in `LearningAPI/views/student_view.py`,
inside the `update` method, placed after the `if "slack_handle" in
request.data` / `if "gitub_handle" in request.data` blocks and before the
`return Response(None, status=status.HTTP_204_NO_CONTENT)` line.

## 5. What the test verifies

`test_patch_slack_handle_persists_to_database` sends a PUT request to update
a student's slack_handle, then calls `nss_user.refresh_from_db()` to force a
fresh read directly from the database (bypassing the in-memory Python object,
which would otherwise still hold the new value even if it was never actually
saved). It then asserts that the refreshed value matches what was sent. This
is the right thing to test because the bug's entire danger was that the view
returned a correct-looking 204 status without actually persisting data; a
test that only checked the response status code would have passed even with
the bug present. Re-reading from the database is what actually proves
persistence happened.