# Bug Fix: Student Assessment Retrieval

## The Bug
A POST to /assessments with only studentId and assessmentId returned a 400 with
"'NoneType' object has no attribute 'user'". The variable that was None was
`obj.instructor` inside StudentAssessmentSerializer.get_instructor_username, which
called `obj.instructor.user.username` unconditionally. The `instructor` field on
StudentAssessment is nullable (null=True), and `create` does not set one, so a newly
created student assessment has no instructor. The error was raised while serializing
the response inside the try block in `create`, so it was caught by `except Exception`
and returned as a 400.

## Log Statements Added
- INFO: "StudentAssessmentView create request received" (data=request.data)
- DEBUG: "create inputs" (book_id, source_url, name, objectives)
- INFO: "StudentAssessmentView create else branch (student assessment)"
- DEBUG: "student assigned", "assessment assigned", "status assigned"
- DEBUG: "student_assessment saved"
- ERROR: "Failed to create student assessment" (error=str(ex), exc_info=True)

## What the Logs Revealed
The ERROR log entry ("Failed to create student assessment") included a stack trace
showing the failure path: create -> serializer.data -> to_representation ->
get_instructor_username, which runs `return obj.instructor.user.username`. The
AttributeError showed that `obj.instructor` was None. The exception was raised inside
transaction.atomic(), so the new record was rolled back and the API returned a 400.

## The Fix
I added a None check in StudentAssessmentSerializer.get_instructor_username so it
returns None when no instructor is assigned. The same request now returns 201 with
"instructor_username": null. I did not assign an instructor in `create`, because the
field is nullable by design, and the requesting user (an admin) has no NssUser record.

## Testing Notes
Tested with studentId 1 and assessmentId 1. Before the fix: 400. After the fix: 201.

