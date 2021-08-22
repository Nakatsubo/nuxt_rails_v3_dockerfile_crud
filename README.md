# Rails API + Nuxt.js on Docker

## 1. Create Directory & Configuration files

Set for Docker image. Create Directory & Configuration files.

```bash
$ touch {docker-compose.yml,.env,.gitignore}
$ mkdir {front,back}
$ cd front
$ touch Dockerfile
$ cd ../back
$ touch {Dockerfile,Gemfile,Gemfile.lock}
```

### front/Dockerfile
Alpine Linux is Linux distributions

```dockerfile
FROM node:16.7.0-alpine

ARG WORKDIR
ARG CONTAINER_PORT

ENV HOME=/${WORKDIR} \
    LANG=C.UTF-8 \
    TZ=Asia/Tokyo \
    HOST=0.0.0.0

# ENV check
RUN echo ${HOME}
RUN echo ${CONTAINER_PORT}

WORKDIR ${HOME}

EXPOSE ${CONTAINER_PORT}
```

### back/Dockerfile

```dockerfile
FROM ruby:3.0.2-alpine

ARG WORKDIR

ENV RUNTIME_PACKAGES="linux-headers libxml2-dev make gcc libc-dev nodejs tzdata postgresql-dev postgresql git" \
    DEV_PACKAGES="build-base curl-dev" \
    HOME=/${WORKDIR} \
    LANG=C.UTF-8 \
    TZ=Asia/Tokyo

# ENV test
RUN echo ${HOME}

WORKDIR ${HOME}

COPY Gemfile* ./

RUN apk update && \
    apk upgrade && \
    apk add --no-cache ${RUNTIME_PACKAGES} && \
    apk add --virtual build-dependencies --no-cache ${DEV_PACKAGES} && \
    bundle install -j4 && \
    apk del build-dependencies

COPY . .

CMD ["rails", "server", "-b", "0.0.0.0"]
```

### .env

```
# Set Environment Variable
WORKDIR=app
CONTAINER_PORT=3000
API_PORT=3000
FRONT_PORT=8080

# db
POSTGRES_PASSWORD=password
```

### .gitignore

```
/.env
```

### docker-compose.yml

```yml
version: '3.8'

services:
  db:
    image: postgres:13.4-alpine
    environment:
      TZ: Asia/Tokyo
      PGTZ: Asia/Tokyo
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
    volumes:
      - ./back/tmp/db:/var/lib/postgresql/data

  back:
    build:
      context: ./back
      args:
        WORKDIR: $WORKDIR
    environment:
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
    volumes:
      - ./back:/$WORKDIR
    depends_on:
      - db
    ports:
      - "$API_PORT:$CONTAINER_PORT"

  front:
    build:
      context: ./front
      args:
        WORKDIR: $WORKDIR
        CONTAINER_PORT: $CONTAINER_PORT
    command: yarn run dev
    volumes:
      - ./front:/$WORKDIR
    ports:
      - "$FRONT_PORT:$CONTAINER_PORT"
    depends_on:
      - back
```

## 2. Creare Docker Image
Launch the Docker app. Create Docker image.

```bash
$ docker-compose build
$ docker images
```

## 3. Create Rails App
Create new Rails App in api mode.

### 3-1. Create new Rails App

```
$ docker-compose run --rm back rails new . -f -B -d postgresql --api

# Rebuilding api directory for update Gemfile
$ docker-compose build back
```

### 3-2. Set Rails DB

```
$ vi back/config/database.yml
```

### back/config/database.yml

```yml
default: &default
   adapter: postgresql
   encoding: unicode
   host: db # add
   username: postgres # add
   password: <%= ENV["POSTGRES_PASSWORD"] %> # add
    ...
```

## 3-3. Create Rail DB

```
$ docker-compose run --rm back rails db:create
```

### Check Rails App

```
# start Rails
$ docker-compose up back

# access this url
http://localhost:3000

# stop Rails
control + c
$ docker-compose ps

# delete container
$ docker-compose rm -f
$ docker-compose ps

# or this command
$ docker-compose down
$ docker-compose ps

-> not found
```

### Set DB passcode

```bash
$ docker-compose up -d db
$ docker-compose exec -u username db psql
```

```bash
postgres=# ALTER USER username WITH PASSWORD 'new passcode';

# check
postgres=# SELECT * FROM pg_shadow;

# quit
postgres=# \q
```

```bash
$ docker-compose down
```

### back/config/database.yml

```yml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: postgres
  password: new passcode # replace
  ...
```

## 4. Create new Nuxt.js App
Create new Nuxt.js App.

```bash
$ docker-compose run --rm front yarn create nuxt-app app
```

```bash
create-nuxt-app v3.7.1
✨  Generating Nuxt.js project in app
? Project name: app
? Programming language: JavaScript
? Package manager: Yarn
? UI framework: None
? Nuxt.js modules: (Press <space> to select, <a> to toggle all, <i> to invert se
lection)
? Linting tools: (Press <space> to select, <a> to toggle all, <i> to invert sele
ction)
? Testing framework: None
? Rendering mode: Single Page App
? Deployment target: Server (Node.js hosting)
? Development tools: (Press <space> to select, <a> to toggle all, <i> to invert
selection)
? What is your GitHub username?
? Version control system: Git
```

```bash
$ mv front/app/{*,.*} front
$ rm -rf front/app
```

### Check Nuxt.js App

```bash
# start Nuxt
$ docker-compose up front

# access this url
http://localhost:8080

# stop Nuxt
control + c
$ docker-compose ps

# delete container
$ docker-compose stop
$ docker-compose rm -f
$ docker-compose ps

# or this command
$ docker-compose down
$ docker-compose ps

-> not found
```

## 4. Create MVC on Rails App
Create User MVC on Rails App.

```bash
$ docker-compose run --rm back rails g scaffold user name:string
$ docker-compose run --rm back rails db:migrate

# check Rails routes
$ docker-compose run --rm back rails routes

Prefix Verb   URI Pattern            Controller#Action
users  GET    /users(.:format)       users#index
       POST   /users(.:format)       users#create
user   GET    /users/:id(.:format)   users#show
       PATCH  /users/:id(.:format)   users#update
       PUT    /users/:id(.:format)   users#update
       DELETE /users/:id(.:format)   users#destroy
```

### Create test user

```bash
# Containers run in the background.
$ docker-compose up -d back
$ curl -X POST http://localhost:3000/users -d 'user[name]=test'
$ curl http://localhost:3000/users/1

curl http://localhost:3000/users/1
{"id":1,"name":"test","created_at":"2021-xx-xxTxx:xx:xx.xxxx","updated_at":"2021-xx-xxTxx:xx:xx.xxxx"}

$ docker-compose down
$ docker-compose ps
```
