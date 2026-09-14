# learn-ops-api: Service Exploration

## 1. Top-level folders in `learn-ops-api`

| Folder                   | Why does this folder need to exist? |
|--------------------------|-------------------------------------|
| .github                  |    GitHub Specific Config,usually CI/CD workflow Actions that run on push/PR                                                                 
| .vscode                  |    Editor-specific configuration shared with team.extensions.json recomends extension for .env file support. launch.json setup remote 
                           |    debugging. mapping local path to to the containers file path so brakpoints  
                           |     workr correctly                                                                                                           
| Learning API             |    Main Django app - this is where actual business logic lives                                
| LearningPlatform         |    it doesn't contain business logic,it contains the configuration that ties everything together,which apps exist,how the database connects,what  
                           |    URL prefixes route to which app and how whole thing gets launches.
| Logs                     |    Log output files
| LogViewer                |    Is a small tool internal tool meant for developers toview logs directly in browser
| config                   |    holds infrastructure/development configuration,seperate from Django's own settings.Specifically nginx reserve-proxy config that route incoming 
                           |    traffic to Django API and React client,plus what looks like a deployment manifest learn-ops-api.yml 
| static                   |    static assets (CSS/JS/images) before Django's collectstatic process them         
| staticfiles              |    statisticfiles/admin-css,js,images for Django's built-in admin site       /framework - css,js,docs,fonts,images for DRF's browsable API -that's the 
                                nice HTML interface DRF gives you when you visit an API endpoint directly in a browser(instead of raw json),letting you test endpoints, see forms, etc.                                
| templates                |    (with base_site.html, base.html, nav_sidebar.html) — these are actually overridden Django admin templates. This is a common Django customization 
                                pattern: you copy the admin app's default templates into your own project so you can tweak branding/layout (e.g. custom site title, logo, nav) without touching Django's internal source code.                                  

## 2. Folders inside `LearningAPI`

| Folder | What responsibility does it own and why? |
|--------|----------------------------------------------------------|
|        |                                                          |
|        |                                                          |
|        |                                                          |

## 3. What is the Pipfile?

Pipfile is used by pipenv ,Python's dependency management tool.It replaces the older requirements.txt pattern(we can see requirements.txt empty).
Declared which packages the project depends on,split into [packages](production) and [dev-packages](only needed for local dev/testing,like pytest)

pip file has * means any version is fine.- it doesn't lock down an exact version by itself.
pipfile.lock - records the exact resolved version that was actually installed,down to the specific path version and even the cryptographic of each package-guaranteeing everyone installing from this lock file gets byte-for-byte identical packages.

## 4. Key packages in the Pipfile

| Package | What functionality does it provide and why? |
|---------|----------------------------------|
| django | |
| djangorestframework | |
| django-allauth | |

## 5. What does `decorators.py` do?


## 6. What is a serializer, and what serializers are defined here?


## 7. Models and what they represent

| Model | Real-world thing it represents |
|-------|-------------------------------|
|       |                               |
|       |                               |
|       |                               |

## 8. Views vs. viewsets

| Type | Example class | File path |
|------|--------------|-----------|
| View | | |
| ViewSet | | |

## 9. Serializers paired with their models

| Serializer | Model | Link |
|------------|-------|------|
|            |       |      |
|            |       |      |

## 10. What replaces the Templates and why?