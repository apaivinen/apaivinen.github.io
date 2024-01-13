---
title: Creating docker image of CyberChef tool
date: 2023-03-04 5:00:00 
categories: [Docker, Creating container image]
tags: [Docker,CyberChef,Creating container image,Self-Hosted,Learning,Tutorial]
layout: post
toc: true
comments: false
math: false
mermaid: false
---

I wanted to learn how to create a basic docker image which contains ready made tool. [CyberChef](https://github.com/gchq/CyberChef) has been on my todo-list to containerize for a while. Yes CyberChef can be run locally but why not to create a container out of it for a selfhosted solution?

Basicially I wanted to create image which does the following
1. Get source code from github
2. Compiles source code for usable form
3. Web server to serve the tool

Fortunately [CyberChef Wiki](https://github.com/gchq/CyberChef/wiki/Getting-started) have a great instructions how to install & compile the tool.

## CyberChef description
Direct quote:
>"The Cyber Swiss Army Knife - a web app for encryption, encoding, compression and data analysis"

Live CyberChef tool at their [Github Pages](https://gchq.github.io/CyberChef/)

# Dockerfile

I already knew nginx would be more than sufficient for my needs so I searched for [nginx container images](https://hub.docker.com/_/nginx) and found out that `nginx:1.23.3-alpine-slim` image is great for this project.

```dockerfile
# Get basic(slim) nginx server image
FROM nginx:1.23.3-alpine-slim as build
```

Next I needed to install Git, NodeJS and NPM to get CyberChef source code and to compile it.

```Dockerfile
# Install git to get cyberchef & tools to build cyberchef
RUN apk update
RUN apk --update add git nodejs npm
```

Next part is just basic get the code and compile it.
```Dockerfile
# Clone and build cyberchef
RUN git clone https://github.com/gchq/CyberChef.git /tmp/cyberchef
WORKDIR /tmp/cyberchef
RUN npm install
RUN npm run build
```
So I'm downloading the source code to `/tmp/cyberchef` where I run the commands found in CyberChef wiki.

Up to this point this is pretty basic stuff. The part which gave me an headache was how to get compiled package to be served in nginx.

First I'm deleting default index.html from ngingx. I'm pretty sure there's a more elegant way to replace default index.html with cyberchefs index.html but this will do for now.
```Dockerfile
# Remove default nginx index.html
RUN rm -f /usr/share/nginx/html/index.html
```

Next I'm going to find compiled .zip package from build output folder and unzip it to `/usr/share/nginx/html/`
```Dockerfile
# Unzip compiled cyberchef zip package
RUN find /tmp/cyberchef/build/prod/ -type f -name "*.zip" | xargs -n1 unzip -d /usr/share/nginx/html/
WORKDIR /usr/share/nginx/html
```

And then I needed to rename `CyberChef_v9.55.0.html` to `index.html` so I don't have to do any modifications to nging configurations.  By default CyberChef naming scheme is `CyberChef_VersionNumber.html` which needs to be changed to `index.html`
```Dockerfile
# Move CyberChef html file to index.html
RUN find . -type f -name "*.html" | xargs -I {FileName} mv {FileName} ./index.html
```

And this could be the last thing to to, to instruct image to start nginx when starting (Spoiler alert, don't use this yet).
```Dockerfile
# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```
As when I was testing this image I noticed it's huge. I mean way too huge.   
Base image is less than 10MB and compiled CyberChef is 31MB. The image I created was more than 600MB in total even after deleting source codes & NodeJS modules. So I needed to figure out a way to shrink the image size.  

Then I found-out about "multi-stage builds".  
Basically I needed to split my image building to two stages.   
Stage one named as build contains all the necessary tools to create the end result which is working CyberChef tool.  
Stage two named main copies CyberChef tool to ngingx to serve.   

![Explanation of two stage build](/assets/img/2023-03-04-creating-cyberchef-image/Two-stage-build.svg)

So this is why when selecting image you can see `FROM nginx:1.23.3-alpine-slim as build` at the first code block. 

Last part of the docker file contains copying CyberChef files from build stage to pristine nging image. Now the image size is a bit over 51MB which is over 10 times less than the original what I created!  
```Dockerfile
# use same nginx alpine slim image as build stage uses
FROM nginx:1.23.3-alpine-slim as main

# Copy build cyberchef files to main for nginx to serve
COPY --from=build /usr/share/nginx/html /usr/share/nginx/html

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```
# Docker-compose file
I'm using docker compose to build and configure this image to container. Docker compose itself is really simple. First there's the necessary build process. Then I'm just naming the container and publishing port 80 from host to port 80 in container.

```
version: '3'
services:
  cyberchef:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: CyberChef
    ports:
      - 80:80
```
# Container usage

Build and create (and start) the container by running command `docker compose up -d`  
Stop the container by running command `docker compose stop`  
Start the container by running command `docker compose start`  
Remove container & network by running command `docker compose down`  

I haven't had a change to test this but in theory when CyberChef is updated on github you could just remove the old image and run `docker compose down` followed by `docker compose up -d`and container should be up-to-date.

# Closing words

There were a lot ot figuring out how to do things allmost on every corner. The end result is just tip of the iceberg if I'm thinking all the things I tried. I'm not a professional on this but in the end I'm pleased to the result which definitely will be in use for work stuff and free time activities.  
This "excercise" gave a bit practice to basic bash commands and how to utilize them and also touch of how to manage npm stuff. Now I can put new containerization project on my todo list for [CISAs MITRE ATT&CK® framework Decoder ](https://github.com/cisagov/decider)

Also here's a [link to my solution in Github](https://github.com/apaivinen/docker/tree/main/dockerfile/cyberchef).

# Sources
I found this article while figuring out how to shring the image to smaller form [How To Reduce Docker Image Size: 5 Optimization Methods (devopscube.com)](https://devopscube.com/reduce-docker-image-size/)  
CyberChef repository [gchq/CyberChef: The Cyber Swiss Army Knife - a web app for encryption, encoding, compression and data analysis (github.com)](https://github.com/gchq/CyberChef)  
CyberChef wiki [Getting started · gchq/CyberChef Wiki (github.com)](https://github.com/gchq/CyberChef/wiki/Getting-started)  
