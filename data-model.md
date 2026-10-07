# Data Model

## 1. Database Diagram

![Diagram](./data-model.png)

## 2. Database Info

PostgreSQL 16 (from docker-compose.yml: `image: postgres:16`)

## 3. Model to Table Mapping

| Model Name | Table Name |
|------------|------------|
| StudentNote|  LearningAPI_studentnote             |
| StudentNoteType        |  LearningAPI_studentnotetype         |

| Property Name | Column Name | Data Type |
|---------------|-------------|-----------|
|      id       |  id         | integer   |
|  note         | note        | text      |
|  created_on   | created_on  | timestamp with zone |
   student        Coach_id      integer
   coach          student_id    integer
   note_type                note_type_id  integer

| Python Field | DB Column |
|--------------|-----------|
| id (implicit) | id    |
| label         | label |

### ORM Write Example (book_view.py)

In `create()`, a new `Book()` object is built up field by field:

```python
book = Book()
book.description = request.data["description"]
book.name = request.data["name"]
book.index = request.data["index"]
book.course = course
book.save()
```

The first four lines just set attributes on a Python object in memory, nothing touches the database yet. The single call to `book.save()` is what the ORM translates into a SQL `INSERT INTO LearningAPI_book (...) VALUES (...)` statement. `book.course = course` is especially worth noting: `course` is itself a `Course` model instance fetched a line earlier, and assigning it to the `course` field is what becomes `course_id = <that course's pk>` in the actual INSERT. The developer never writes SQL directly; setting `.course` to an object is enough for the ORM to store the matching foreign key value.

## 4. Relationship Examples

**One-to-one** (field name: )

| Model Name | Table Name | PK Column | FK Column |
|------------|------------|-----------|-----------|
| NssUser | LearningAPI_nssuser | id | user_id |
| User (Django built-in) | auth_user | id | |

LearningAPI/models/people/nssuser.py
`user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)`
Each NssUser links to exactly one Django auth User, and each User links back
to at most one NssUser.



**One-to-many** (field name: )

| Model Name | Table Name | PK Column | FK Column |
|------------|------------|-----------|-----------|
| NssUser | LearningAPI_nssuser | id | |
| StudentNote | LearningAPI_studentnote | id | coach_id |

LearningAPI/models/people/student_note.py
`coach = models.ForeignKey(NssUser, on_delete=models.CASCADE)`
One NssUser (acting as coach) can be linked from many StudentNote rows, each
note points back to exactly one coach via coach_id.


**Many-to-many** (field name: )

| Model Name | Table Name | PK Column | FK Column |
|------------|------------|-----------|-----------|
| StudentTeam | LearningAPI_studentteam | id | |
| NssUser | LearningAPI_nssuser | id | |
| (junction) NSSUserTeam | LearningAPI_nssuserteam | id | team_id, student_id |


LearningAPI/models/people/student_team.py
`students = models.ManyToManyField("NSSUser", through="NSSUserTeam")`
A StudentTeam can have many students, and a student (NssUser) can belong to
many teams. The junction table LearningAPI_nssuserteam makes this possible: it
has no fields of its own beyond two foreign keys, team_id (→ StudentTeam) and
student_id (→ NssUser), one row per student-team pairing.