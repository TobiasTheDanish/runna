# Runna

### Description

Runna is a personal project of mine, where i want to be able to track my running sessions.

I want to be able to create new "sessions" where i input relevant information about a run.

I also want a way to get an overview of my registered sessions.

### Requirements

Below is a list of requirements for the project

- A json api backend written in Go/golang
  - Separate git repo in a directory within the same directory as this file
  - Needs to be runnable from docker with docker compose.
  - Libsql database with turso as the database provider. You can assume an env variable with db url.
  - Endpoint for creating a new "session".
  - Endpoint for getting "sessions" within a given time period.
- A javascript frontend using svelte/sveltekit
  - Separate git repo in a directory within the same directory as this file
  - Needs to be runnable from docker with docker compose.
  - UI for creating a new "session".
  - UI for display "sessions" within a given time period.
