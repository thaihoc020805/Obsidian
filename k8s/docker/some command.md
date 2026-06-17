version
docker version (check version docker)

docker pull image-name (pull existing image from docker hub)
docker run image-name command  (run a container from a image, it will detect that if image was pulled, and pull the image first if it wasn't pulled)

ps
docker ps (list all running containers and some basic information about them)
docker ps -a (list all containers that were stopped or are running)

docker stop container-name/id (stop a container and can run them again)
docker start container-name/id (running a stopped container)
docker restart container-name/id (restart a running container)

docker rm container-name/id (remove a container)


docker images (see all images were pulled)

docker rmi image-name (remove a image)

if container don't run any process, or no longer run any process => it stop
so if you run the comman : docker run ubuntu => it stop immediatly because it just a os, a base image it don't run any process, any services

exec
docker exec container-name/id command (execute a command on a container)

-d
docker run -d image-name (create a container and make it cli run in background)
docker attack container-name/id (attack it cli in your console)

tag:
redis:4.0 hay redis:buster, redis:5.0-alpine -> this is tag
if you don't specific tag of the image, it will automatly choose latest
docker run redis = docker run redis:latest

stdin
docker run -it image-name (i : interactive mode for type from your keyboard, t : like open terminal for container)

port mapping

![[Pasted image 20250727153953.png]]


volume
![[Pasted image 20250727224822.png]]


more than ps
docker inspect docker-name/id (if you want more detail about a container)



logs
(when detach mode)
docker log container-id/name 

## Running with Environment Variables

There are two main ways to pass environment variables to a container:

PS: Dont forget to build the image before running it!

1. Using the `-e` flag:

```
docker run -e SECRET_MESSAGE="Hello, Docker!" -e PORT=8080 my-app
```

You might not want to leak your secrets to the command line, so you can also use a `.env` file:

2. Using the `--env-file` flag:

Create a file named `.env`:

```
SECRET_MESSAGE="Hello, Docker!"
PORT=8080
```

Run the container with the `--env-file` flag:

```
docker run --env-file .env my-app
```

Internally, Docker will insert the environment variables into the container at runtime, like normal process environment variables. During the build process, the environment variables are not available.


## ==Build Arguments vs Environment Variables==

There’s another way to configure Docker: build arguments. The key difference is:

- ==Build args are only available during image build==
- ==Environment variables are available at runtime==

Build arguments are made to configure the build process. For example, you might want to compile your application slightly differently for production and development. Let’s see an example.

Modify the Dockerfile:

``` dockerfile
FROM golang
COPY . .
ARG IS_PRODUCTION=false
# if its production, add a compilation flag
RUN if [ "$IS_PRODUCTION" = "true" ]; then go build -o main main.go -ldflags "-s -w"; else go build -o main main.go; fi
CMD ["./main"]
```

In this Dockerfile we add

```
ARG IS_PRODUCTION=false
```

This defines that the build argument `IS_PRODUCTION` is set to `false` by default. Build the image with the build argument:

```
docker build -t my-app  --build-arg IS_PRODUCTION=true .
```

If we specify `IS_PRODUCTION=true`, the build will execute the `if` statement and add the `-ldflags "-s -w"` flag to the build command. If we don’t specify it, it will use the default value (`false`) and not add the flag.

Now is a great time to think about your usecases. What would you use build arguments for? What would you use environment variables for? Why did you choose one over the other? An example can only show you so much, try it out! If you’re not sure, feel free to message me :)