# Data Model AI Prompts

## 1. Find the Database Connection Details

Host: localhost, Port: 5433, DB: learningplatform, User: learnops
(from learn-ops-infrastructure/.env and docker-compose.yml)

Matched my manual findings exactly. Claude additionally explained why the API's
.env has different values (database:5432, used only inside Docker's internal
network) versus pgAdmin's (localhost:5433, used from the host machine), and
flagged that learn-ops-api/.env contains a real GitHub token and Slack bot
token that must never be shared publicly, something I hadn't noticed looking
at that file myself.

## 2. Identify the Database Type

What database engine does this application use?
Find where that is declared in the project files and tell me the version.


## 3. Map the ORM to the Database
How does this application communicate with the database? Find the ORM it uses
and where the database connection is configured. Include the ENGINE value from settings.

Then open LearningAPI/models/ and pick one model. Show me the Python field names
next to the SQL column names and their data types.

Finally, open LearningAPI/views/book_view.py and find the create method.
What does book.save() actually do - what SQL statement does the ORM generate?

## 4. Generate a Database Diagram
Generate an entity-relationship diagram of the database as a Mermaid erDiagram.
Look at all the model files in LearningAPI/models/.
Include every table, every field, and every relationship between tables.
Output only the Mermaid code block.

## 5. Find Relationship Examples
Find one example each of a one-to-one, one-to-many, and many-to-many relationship
in the Django models in LearningAPI/models/.
For each example give me the file path and the name of the field that defines the relationship.