
# Debug AI Prompts

## Step 2 Prompt

The Slack handle update feature has a bug. When a user updates their Slack
handle through the student profile, the API returns a success response
(204 No Content), but the change is never actually saved to the database. A
subsequent GET request still shows the old value.

Read LearningAPI/views/student_view.py, specifically the update method, and
explain exactly why this is happening. Be specific about which line is
missing or incorrect, and cite the exact line number.

## Step 4 Prompt

Write the test method test_patch_slack_handle_persists_to_database in
LearningAPI/tests/test_student_update.py. The test class StudentPartialUpdateTests
already sets up a user, a token, and an NssUser with slack_handle='@old_handle'
in its setup. The test currently just contains fail("Not Implemented yet").

The test should:
1. Make a PUT request to /students/{pk}/ with a new slack_handle value.
2. Assert the response status is 204.
3. Re-fetch the NssUser from the database.
4. Assert the re-fetched record's slack_handle matches the value sent.

Show me the test code but do not edit the file yourself.

## Step 6 Prompt
I understand the bug now: student.save() is missing in the update method of
LearningAPI/views/student_view.py. What single line do I need to add, and
exactly where in the method does it belong? I want to type the fix myself,
so just tell me the line and its exact location, don't edit the file.