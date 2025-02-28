---
title: Use containers for Node.js development
keywords: node, node.js, development
description: Learn how to develop your Node.js application locally using containers.
aliases:
- /get-started/nodejs/develop/
---

## Prerequisites

Complete [Containerize a Node.js application](containerize.md).

## Overview

In this section, you'll learn how to set up a development environment for your containerized application. This includes:
 - Adding a local database and persisting data
 - Configuring your container to run a development environment
 - Debugging your containerized application

## Add a local database and persist data

You can use containers to set up local services, like a database. In this section, you'll update the `compose.yaml` file to define a database service and a volume to persist data.

Open the `compose.yaml` file in an IDE or text editor. You'll notice it
already contains commented-out instructions for a Postgres database and volume.

Open `src/persistence/postgres.js` in an IDE or text editor. You'll notice that
this application uses a Postgres database and requires some environment
variables in order to connect to the database. The `compose.yaml` file doesn't
have these variables defined.

You need to update the following items in the `compose.yaml` file:
 - Uncomment all of the database instructions.
 - Add the environment variables under the server service.
 - Add `secrets` to the server service for the database password.

The following is the updated `compose.yaml` file.

```yaml
services:
  server:
    build:
      context: .
    environment:
      NODE_ENV: production
      POSTGRES_HOST: db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password
      POSTGRES_DB: example
    ports:
      - 3000:3000
    depends_on:
      db:
        condition: service_healthy
    secrets:
      - db-password
  db:
    image: postgres
    restart: always
    user: postgres
    secrets:
      - db-password
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=example
      - POSTGRES_PASSWORD_FILE=/run/secrets/db-password
    expose:
      - 5432
    healthcheck:
      test: [ "CMD", "pg_isready" ]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  db-data:
secrets:
  db-password:
    file: db/password.txt
```

> **Note**
>
> To learn more about the instructions in the Compose file, see [Compose file
> reference](/compose/compose-file/).

Before you run the application using Compose, notice that this Compose file uses
`secrets` and specifies a `password.txt` file to hold the database's password.
You must create this file as it's not included in the source repository.

In the cloned repository's directory, create a new directory named `db`. Inside the `db` directory, create a file named `password.txt`. Open `password.txt` in an IDE or text editor and add a password of your choice. The password must be on a single line with no additional lines in the file.

You should now have the following contents in your `docker-nodejs-sample`
directory.

```text
├── docker-nodejs-sample/
│ ├── db/
│ │ └── password.txt
│ ├── spec/
│ ├── src/
│ ├── .dockerignore
│ ├── .gitignore
│ ├── compose.yaml
│ ├── Dockerfile
│ ├── package-lock.json
│ ├── package.json
│ └── README.md
```

Run the following command to start your application.

```console
$ docker compose up --build
```

Open a browser and verify that the application is running at [http://localhost:3000](http://localhost:3000).

Add some items to the todo list to test data persistence.

After adding some items to the todo list, press `ctrl+c` in the terminal to stop your application.

In the terminal, run `docker compose rm` to remove your containers and then run `docker compose up` to run your application again.

```console
$ docker compose rm
$ docker compose up --build
```

Refresh [http://localhost:3000](http://localhost:3000) in your browser and verify that the todo items persisted, even after the containers were removed and ran again.

## Configure and run a development container

You can use a bind mount to mount your source code into the container. The container can then see the changes you make to the code immediately, as soon as you save a file. This means that you can run processes, like nodemon, in the container that watch for filesystem changes and respond to them. To learn more about bind mounts, see [Storage overview](../../storage/index.md).

In addition to adding a bind mount, you can configure your Dockerfile and `compose.yaml` file to install development dependencies and run development tools.

### Update your Dockerfile for development

Open the Dockerfile in an IDE or text editor. Note that the Dockerfile doesn't
install development dependencies and doesn't run nodemon. You'll
need to update your Dockerfile to install the development dependencies and run
nodemon.

Rather than creating one Dockerfile for production, and another Dockerfile for
development, you can use one multi-stage Dockerfile for both.

Update your Dockerfile to the following multi-stage Dockerfile.

```dockerfile
# syntax=docker/dockerfile:1

ARG NODE_VERSION=18.0.0

FROM node:${NODE_VERSION}-alpine as base
WORKDIR /usr/src/app
EXPOSE 3000

FROM base as dev
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=cache,target=/root/.npm \
    npm ci --include=dev
USER node
COPY . .
CMD npm run dev

FROM base as prod
ENV NODE_ENV production
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev
USER node
COPY . .
CMD node src/index.js
```

In the Dockerfile, you first add a label `as base` to the `FROM
node:${NODE_VERSION}-alpine` statement. This allows you to refer to this build
stage in other build stages. Next, you add a new build stage labeled `dev` to
install your dev dependencies and start the container using `npm run dev`.
Finally, you add a stage labeled `prod` that omits the dev dependencies and runs
your application using `node src/index.js`. To learn more about multi-stage
builds, see [Multi-stage builds](../../build/building/multi-stage.md).

Next, you'll need to update your Compose file to use the new stage.

### Update your Compose file for development

To run the `dev` stage with Compose, you need to update your `compose.yaml` file.
Open your `compose.yaml` file in an IDE or text editor, and then add the
`target: dev` instruction to target the `dev` stage from your multi-stage
Dockerfile.

Also, add a new volume to the server service for the bind mount. For this application, you'll mount `./src` from your local machine to `/usr/src/app/src` in the container.

Lastly, publish port `9229` for debugging.

The following is the updated Compose file.

```yaml
services:
  server:
    build:
      context: .
      target: dev
    environment:
      NODE_ENV: production
      POSTGRES_HOST: db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password
      POSTGRES_DB: example
    ports:
      - 3000:3000
      - 9229:9229
    depends_on:
      db:
        condition: service_healthy
    secrets:
      - db-password
    volumes:
      - ./src:/usr/src/app/src
  db:
    image: postgres
    restart: always
    user: postgres
    secrets:
      - db-password
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=example
      - POSTGRES_PASSWORD_FILE=/run/secrets/db-password
    expose:
      - 5432
    healthcheck:
      test: [ "CMD", "pg_isready" ]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  db-data:
secrets:
  db-password:
    file: db/password.txt
```

### Run your development container and debug your application

Run the following command to run your application with the new changes to the `Dockerfile` and `compose.yaml` file.

```console
$ docker compose up --build
```

Open a browser and verify that the application is running at [http://localhost:3000](http://localhost:3000).

Any changes to the application's source files on your local machine will now be
immediately reflected in the running container.

Open `docker-nodejs-sample/src/static/js/app.js` in an IDE or text editor and update the button text on line 109 from `Add Item` to `Add`.

```diff
+                         {submitting ? 'Adding...' : 'Add'}
-                         {submitting ? 'Adding...' : 'Add Item'}
```

Refresh [http://localhost:3000](http://localhost:3000) in your browser and verify that the updated text appears.

You can now connect an inspector client to your application for debugging. For
more details about inspector clients, see the [Node.js
documentation](https://nodejs.org/en/docs/guides/debugging-getting-started).

## Summary

In this section, you took a look at setting up your Compose file to add a mock
database and persist data. You also learned how to create a multi-stage
Dockerfile and set up a bind mount for development.

Related information:
 - [Volumes top-level element](/compose/compose-file/07-volumes/)
 - [Services top-level element](/compose/compose-file/05-services/)
 - [Multi-stage builds](../../build/building/multi-stage.md)

## Next steps

In the next section, you'll learn how to run unit tests using Docker.

{{< button text="Run your tests" url="run-tests.md" >}}
