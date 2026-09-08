# Tech Stack AI Prompts

## 1a. Config Files
Find all config and environment files in the learn-ops-infrastructure repo, and create a markdown table in tech-stack-ai.md with columns Config File, Location, Config Value, What it's for, How it's used. Include at least 3 values per file. Use the config variable name only — never the literal value (e.g. write POSTGRES_USER, not the actual password).

## 1b. How to Start It 
Look at the Makefile at the root of the learn-ops-infrastructure repo and document how to start the system in tech-stack-ai.md. Identify which targets are relevant to starting the system and explain how they differ from each other.

## 1c . Where to Access It
Find each service's port and URL by checking the relevant config files in this repo, and add a markdown table to tech-stack-ai.md with columns Service, Port, URL.

## 1d . Service Dependencies
Map the service dependencies in this system and add a markdown table to tech-stack-ai.md with columns Service, Depends On, Why. For the Why column, explain how the dependent service actually uses the other service — not just that a dependency exists.

## 1e . Main Entry Points
For each service, find the file where the process starts (the startup file) and the file where incoming requests/routes are defined (routes or URL config). These should be two different files per service. Add a markdown table to tech-stack-ai.md with columns Service, Startup File, Routes / URL Config File.

## Part 2: Document the Services
Add a Services section to tech-stack-ai.md with a markdown table with columns Service Name, Tech Stack (including version), and Purpose — one row per service in this system.

## Good — now Part 3: System Overview
Add a System Overview section to tech-stack-ai.md with 3 paragraphs: the first describing what kind of application this is and what problem it solves (not the tech, just the purpose), the second describing its main features from a user's perspective — what can they do, what does a typical session look like, and the third describing who uses it and whether different roles interact with it differently.