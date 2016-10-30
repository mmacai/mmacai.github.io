---
published: true
title: Let's become friends with Docker by practice
layout: post
tags: [docker, nodejs]
categories: [docker, nodejs]
---
# Part 1

## Contains
- Setup Dockerfile
- Images
  - Build
  - Tags
  - Push
- Containers
  - Run
  - Stop, Start
  - Links & Networks
  - Exec
  - Inspect
  - Volumes
  - Remove
  - Logs
  - Environment variables
  - External commands
- Docker compose
  - Build
  - Start, Stop & Remove

## Brief overview / Introduction
Docker is an open-source tool designed to create, build, deploy and run applications as easy as possible using containers. You are able to create system with all necessary things you need and wrap your application with it. This way environment is always the same. Unlike virtual machines, Docker shares OS kernel across containers, where every container is running as a process on this hosted OS.

## Let’s practice already
Before we can start using Docker, we need to install it. Following link contains all needed information. Just select type of your OS, download and install.  
https://www.docker.com/products/overview

## Dockerfile
Now that Docker is installed, we are ready to start. Docker needs to have some sort of configuration to be able to build applications. Entry point file with this configuration is called `Dockerfile`. Yes its just `Dockerfile`, no extension, nothing. It should be placed in the root directory of given application.
However structure of content of this file may vary, in most cases there is some operations order. Below is example how can `Dockerfile` looks like.

For practice I prepared two repositories/apps using NodeJS you can use. Only thing you need is to git clone them and run `npm i` inside both of them.

[App](https://github.com/mmacai/blog-example-app.git)  
[Service](https://github.com/mmacai/blog-example-service.git)

Both of them contain prepared `Dockerfile`. You can try to run app and service using `npm start` command.

`http://localhost:4000`  
`http://localhost:5000`  
`http://localhost:5000/service`

Below is described app's `Dockerfile`(second one is similar).

```
# 1 Environment
FROM mhart/alpine-node:6.5

# 2 Attach files to container
COPY index.js /node-app/index.js
COPY package.json /node-app/package.json
COPY README.md /node-app/README.md

# 3  Working directory
WORKDIR /node-app/

# 4 Install all dependencies
RUN npm install

# 5 Volume example
RUN mkdir /myvol
RUN echo "hello world" > /myvol/greeting.txt
VOLUME /myvol

# 6 Health check
HEALTHCHECK --interval=2m --timeout=3s \
  CMD curl -f http://localhost:4444/ || exit 1

# 7 Environment variable example
ENV PORT 4444

# 8 Expose ports
EXPOSE ${PORT}

# 9 Start app
CMD ["npm", "start"]
```

`# 1` describes what environment you want to use. In this scenario we have NodeJS application, so we want to use NodeJS environment. Most of the time all you need is just search in google for your desired environment (Java, Python, Ruby, etc.) and add docker word to it. There are plenty of containers for almost everything you can imagine. Here using NodeJS we will be able to use NodeJS commands (respective to chosen NodeJS version).

`# 2` specifies what files should be attached to container. Path to these files are based on position of `Dockerfile`, so be aware of that. You can also use `ADD` instead of `COPY`, but `COPY` looks more transparent.

`# 3` says what working directory you want to use. It's not required, but good to use so your application is stored on place you can easily find(inside container).

`# 4` just runs `npm install` to install all dependencies as you would do locally not using docker.

`# 5` shows example how volumes work. Basically you will create new empty directory, create file with text `hello world` and mount this folder. This is just simple example, in most cases you will use volumes for databases etc. In `Dockefile` you can only mount container's content which will be automatically mapped to docker root directory(`docker run` can bind volumes to custom host folder).

`# 6` sets health checks for image. As options are saying it will occur every 2 minutes with timeout of 3 seconds. Second is condition at which it will be evaluated, so if this fails container will be considered unhealthy. For now health check will fail, because NodeJS distribution we are using doesn't have curl. But in this place you should place whatever you think is telling about your container health status. For example when database is ready etc.

`# 7` sets environment variable for port. If not specified, app will run on port 5000, but using this we will overwrite it.

`# 8` says that application will be available on port 4444 set by environment variable. Which means you have to add this port to url when running containerized application.

Finally `# 9` tells docker what should be executed when everything is done. In this case how to start the application.

## Images

### Build

Next step is to build image using our `Dockerfile`. In terminal type:  
`docker build -t=my-app .` (inside app)  
`docker build -t=my-service .` (inside service)

**Don’t forget about dot at the end.** After few seconds/minutes, image is ready to run in container. Make sure that images exist using command: `docker images`

### Tags

Once image exists you are able to tag your image. This option is mostly used for versioning your images. For example version image for production, develop, testing etc. Command would look like this(`my_name` **should be your dockerhub username, see Push section**):
`docker tag my-app my_name/my-app:production`

List images again and you should see now two versions of your app.

### Push

For this you will need account at `dockerhub.com`, so go ahead and register, it's for free. Once you have account we can try to push image we just built and tagged few minutes ago. In terminal run:

`docker login`

You will be asked for account name and password and if everything went well, you will get success message. Only thing that left is to push image up.

`docker push my_name/my-app`

When uploading is done, visit your dockerhub profile and you should see image there. From this point everyone can use it if it remains public. You can also setup private images. To test if it really works. Remove image we just created:

`docker rmi my_name/my-app:production`

and run:

`docker pull my_name/my-app:production`

Image should be downloaded and able to run in container. Check with:

`docker images`

## Containers

### Run

Images are ready to run inside container, so let's go for it:  
`docker run --name my-service -p 4000:4000 -d my-service`
Name of your container can be absolutely different from image name, but it's good to keep it at least similar for easier searching later.

Using this command we have running service locally inside docker. Let’s check it's true with command:
`docker ps`.
Running container should be listed there.

You can also list containers that exists, but are not running:
`docker ps -a`

Container is running and we are finally able to test if it really works. Navigate to `http://localhost:4000` and see if app is working. App should print `Another service, yay!` in browser.

### Stop, Start

Container can be stopped by:
`docker stop my-service`
And started up again:
`docker start my-service`

### Links & Networks

In most cases application is built out of multiple pieces. There is a big chance that some of them are dependent on other ones. To accomplish connection between them you can choose two ways how to do that. In our case we have running service, but didn't start app image yet.

#### Link

Using link flag you can specify on which containers is  dependent or needs to be aware of them. It’s pretty straightforward.

`docker run --name my-app -p 4444:4444 --link my-service:my-service -d -e SERVICE_URL="http://my-service:4000" my-app`

Link flag is very similar to port. First parameter is name of dependency your container expects and second one name of existing container. By doing this container is aware of `my-service` and can communicate with it. However this approach is very common, in these days is obsolete and you should use second approach.

Use `docker ps` and you should see health check info inside braces. Currently it should be at **starting** state.

#### Network

As name says we will create network. It’s pretty self-explanatory. They behave, as you would expect, everything that is on the same network is visible and everything that is on other networks is not. In other words if you have container A on network 1 and container B on network 2. They are not aware about themselves. But what you can do is specify multiple networks for single container and build some kind of network structure.

First you have to create network:
`docker network create my-network`

Stop and remove running containers:
`docker stop my-app my-service && docker rm my-app my-service`

And then place containers on it.  
`docker run --name my-service -p 4000:4000 --network my-network -d my-service`  
`docker run --name my-app -p 4444:4444 --network my-network -d -e SERVICE_URL="http://my-service:4000" my-app`

Visit:

`http://localhost:4000`  
`http://localhost:4444`  
`http://localhost:4444/service`

### Exec

Once container is running you are able to get into it via bash command line(or other variants) and do actions. For example to run shell scripts that are inside container.

`docker exec –it my-app ash` or `docker exec -it my-app /bin/sh`

Now you are inside container and by typing `ls` you should see app structure. If we go one level up by `cd ..` and again `ls` you should see here our mounted folder `myvol` and inside it `greeting.txt` file.

To make our health checks work we need to install curl inside our container. To accomplish that firstly check that curl is not working. Do
`curl` in terminal it should reply that curl was not found. To install curl run:

`apk add --update curl`

Now try `curl --help` and you should see plenty of text with options. That's all for this part exit with `exit` command.

### Inspect

You can access detailed container or image information such as networks, volumes etc. by command:
`docker inspect my-app`

At the start of log is also Health status information. There should be messages in `log` that curl was not found. If you wait for the next health check (2 minutes) and do `docker ps`. Health status should be now set to `healthy`.

As a shortcut to get Health info from inspect you can also use following command:

`docker inspect --format='{{json .State.Health}}' my-app`

You can recognize that healthy status was triggered by success containing `Hello world!` which is our welcome phrase in browser landing page.

### Volumes

If you need to mount your docker container data to disk to not loss them when container is destroyed you are able to use volumes. Easiest example of using it is database. Imagine you run database and there are already some data, but when you remove container data are lost. Volumes will mount data from the container to disk and once you remove container and start it again database will load those data from disk.

In our example we can do something like this(instead of `$(pwd)` you may use any directory you want): 

**For Windows users: Use either GitBash or transform volumes host path($(pwd)/blog-volume) to Windows path style**
 
`docker run --name my-app -p 4444:4444 --network my-network -d -e SERVICE_URL="http://my-service:4000" -v $(pwd)/blog-volume:/myvol2 my-app`

This means that `myvol` folder will be mapped to disk at our current place and put inside `blog-volume`. Stop and remove container and by using command above you should have empty `blog-volume` folder at your current place. We didn't use `myvol`, because it would overlap original mounting from `Dockerfile` and content inside container's `myvol` would be hidden. Let's create some txt file inside `blog-volume` and check if it was added inside container.

`docker exec -it my-app ash`  
`cd ../myvol2`  
`cat name_of_your_file`

Content should be printed in console.

### Remove

If you don’t need any image or container anymore you can remove them simply by:  
`docker rm my-app`  
`docker rmi my-app`

### Logs

Often you want to trace you running app inside container. For example when app goes down for some reason you want to know what happened. Be aware that these logs are just as good as your app is outputting to terminal when you run it locally. Docker will not enhance it with anything.

Given command will give you live logs for chosen container. If don’t want live capability just omit `-f` flag before container name/id.
`docker logs –f my-app`

### Environment variables

As you might see few moments ago we used `-e` flag inside our container run command. That's way how to set environment variables for app when you don’t want to expose things like API keys inside code etc.

`docker run --name my-app -p 4444:4444 --network my-network -d -e SERVICE_URL="http://my-service:4000" my-app`

Quotes around value of environment variables shouldn’t be necessary, but when you assign multiple values to variable as it is shown below it’s easier to read it.

`docker run -- name my-app - e API_KEY=”123456,4567,09876” my-app`

### External commands

Sometimes you may want to run some external command during `docker run`. For this purpose you can extend run command which will overwrite original command from dockerfile. Considering run command we used before:

`docker run --name my-service -p 4000:4000 -d my-service`

We can try following example:

`docker run --name my-service -p 4000:4000 my-service echo Hello && npm start`

It should print `Hello` to terminal output and after that start app. If we omit `npm start` app won't start, don't forget by doing this we are **overwriting dockerfile command**(at the end of the file).

## Docker compose

As app starts to grow up you may find starting every container one by one exhausting. For this purpose exists Docker compose. Docker compose file allows you to specify all containers you want to start with same properties as you are able to use by standard docker run command. Currently are available two versions and we will use the second one(`version: '2'`).

```
version: '2'
services:
    my-app: #1 Reference name
        image: my-app #2 Image name
        container_name: my-app #3 Container name
        environment:
            SERVICE_URL: http://my-service:4000 #4 Environment variable
        ports:
            - 5000:4444 #5 Exposing ports

    my-service:
        image: my-service
        container_name: my-service
        ports:
            - 4000:4000

```

`# 1` says using which name you can reference container inside `docker-compose` file.

`# 2` and `# 3` are pretty self-explanatory, you are telling which image to use and how should be container named when it will be started. In `image` you can also specify url if it's saved somewhere remotely or just put there name of image that is placed in dockerhub. Possible option is also to specify `Dockerfile` and `docker-compose` will build images for you.

`# 4` specifies values to environment variables in given application.

`# 5` is just telling on which port will application be available. You should expose only those containers that you consider safe. Since `docker-compose` is composing containers for you they know about each other.

Very common case is when one container depends on another one. Which means you can specify correct order of containers on boot up using following option:

```
my-app:
    image: my-app
    container-name: my-app
    depends_on:
        - my-db
        - my-service2
```

### Start, Stop & Remove

Since we have docker-compose file we are able to start app. Just by using command:
`docker-compose up –d`

Visit application at: `localhost:5000`, `localhost:4000` and `localhost:5000/service`.

When you run `docker ps` you will see that `docker-compose` started all containers, as you normally would do manually.

If you want to see logs from all containers inside (similar to docker logs command) just omit `-d` flag or use: `docker-compose logs`. In case you want multiple `docker-compose` files (e.g. for development, production, etc.), you are able to specify it using flag `-f`.  
`docker-compose –f different_docker_compose_file.yml up -d`

In same fashion as start you can use stop, rm and kill commands. Which means they will be applied to all containers inside docker compose, so you don’t have to do that one by one (but still can).  
`docker-compose stop`  
`docker-compose rm`  
`docker-compose kill`

Let's try second `docker-compose` file which omits port expose for `my-service`.
`docker-compose -f docker-compose-no-expose.yml up -d`

Application should behave same as before however now you shouldn't be able to visit `localhost:4000`.

These are just most common things Docker offers. For more information visit [Docker Docs](https://docs.docker.com/engine/reference/). In second part we will look at Docker swarm and orchestrating containers.

**Next time: Part 2 - Docker swarm & orchestrating containers.**