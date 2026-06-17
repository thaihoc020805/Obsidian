## Introduction

Docker just released [Docker Compose Watch](https://docs.docker.com/compose/file-watch/) with [Docker Compose Version 2.22](https://docs.docker.com/compose/release-notes/#2220). With this new feature, you can use `docker-compose watch` instead of `docker-compose up` and automatically synchronize your local source code with the code in your Docker container without needing to use volumes!

Let us take a look at how this works in a real-word project by using a [project](https://github.com/Code42Cate/hackathon-starter) that I [previously wrote about](https://dev.to/code42cate/how-to-win-any-hackathon-3i99).

In this project, I have a monorepo with a frontend, backend, and some additional libraries for the UI and database.  

```
├── apps
│   ├── api
│   └── web
└── packages
    ├── database
    ├── eslint-config-custom
    ├── tsconfig
    └── ui
```

Both apps (`api` and `web`) are already dockerized and the Dockerfiles are in the root of the project ([1](https://github.com/Code42Cate/hackathon-starter/blob/main/api.Dockerfile), [2](https://github.com/Code42Cate/hackathon-starter/blob/main/web.Dockerfile))

The `docker-compose.yml` file would look like this:  

```
services:
  web:
    build:
      dockerfile: web.Dockerfile
    ports:
      - "3000:3000"
    depends_on:
      - api
  api:
    build:
      dockerfile: api.Dockerfile
    ports:
      - "3001:3000"from within the Docker network
```

That's already pretty good, but as you already know it's a PITA to work with this during development. You will have to rebuild your Docker images whenever you change your code, even though your apps will probably support hot-reloading out of the box (or with something like [Nodemon](https://www.npmjs.com/package/nodemon) if not).

To improve this, Docker Compose Watch [introduces a new attribute](https://docs.docker.com/compose/file-watch/#configuration) called `watch`. The watch attribute contains a list of so-called **rules** that each contain a **path** that they are watching and an **action** that gets executed once a file in the path changes.

## [](https://dev.to/code42cate/say-goodbye-to-docker-volumes-j9l#sync)Sync

If you would want to have a folder synchronized between your host and your container, you would add:  

```
services:
  web: # shortened for clarity
    build:
      dockerfile: web.Dockerfile
    develop:
      watch:
        - action: sync
          path: ./apps/web
          target: /app/apps/web
```

Whenever a file on your host in the path `./apps/web/` changes, it will get synchronized (copied) to your container to `/app/apps/web`. The additional app in the target path is required because this is our `WORKDIR` defined in the [Dockerfile](https://github.com/Code42Cate/hackathon-starter/blob/main/web.Dockerfile). This is the main thing you will probably use if you have hot-reloadable apps.

## [](https://dev.to/code42cate/say-goodbye-to-docker-volumes-j9l#rebuild)Rebuild

If you have apps that need to be compiled or dependencies that you need to re-install, there is also an action called **rebuild**. Instead of simply copying the files between the host and the container, it will rebuild and restart the container. This is super helpful for your npm dependencies! Let's add that:  

```
services:
  web: # shortened for clarity
    build:
      dockerfile: web.Dockerfile
    develop:
      watch:
        - action: sync
          path: ./apps/web
          target: /app/apps/web
        - action: rebuild
          path: ./package.json
          target: /app/package.json
```

Whenever our package.json changes we will now rebuild our entire Dockerfile to install the new dependencies.

## [](https://dev.to/code42cate/say-goodbye-to-docker-volumes-j9l#syncrestart)Sync+Restart

Besides just synchronizing and rebuilding there is also something in between called sync+restart. This action will first synchronize the directories and then immediately restart your container without rebuilding. Most frameworks usually have config files (such as `next.config.js`) that can't be hot-reloaded (just sync isn't enough) but also don't require a slow rebuild.

This would change your compose-file to this:  

```
services:
  web: # shortened for clarity
    build:
      dockerfile: web.Dockerfile
    develop:
      watch:
        - action: sync
          path: ./apps/web
          target: /app/apps/web
        - action: rebuild
          path: ./package.json
          target: /app/package.json
        - action: sync+restart
          path: ./apps/web/next.config.js
          target: /app/apps/web/next.config.js
```