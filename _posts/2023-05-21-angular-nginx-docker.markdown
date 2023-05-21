---
layout: post
title:  "Angular + NGINX + Docker"
date:   2023-05-15 05:00:00 +0100
published: true
comments: true
categories: Angular Docker
cover: "assets/angular-nginx-docker/angular+nginx+docker"
tags: [Angular, Docker]
type: article
---

We can deploy our Angular applications in many ways, one of them is building the Angular app in an NGINX Docker container and then deploy it anywhere (on an infrastructure that supports Docker at least).

In this article, we'll cover how to build a Docker image that runs our Angular application in NGINX. I assume you already have an existing Angular application. If not you can create one with the following command:

```bash
ng new my-angular-app
```

We start by creating a new folder for our NGINX configuration. 

Create a folder structure `build/nginx` in the root of your Angular project (or name it however you want 😄) with a file called `default.conf` in it (this name you can't choose).
The folder structure should look like this:
```bash
my-angular-app
├── build
│   └── nginx
│       └── default.conf
```

Copy the following content inside of the `default.conf` file:

```editorconfig
# Don't send the nginx version number in error pages and server header
server_tokens off;

gzip on;
gzip_comp_level 5;
gzip_min_length 256;
gzip_proxied any;
gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/javascript application/json application/xml;
       
location ~* / {
  try_files $uri $uri/ /index.html;
}

location ~* \.(?:css|js|jpg|jpeg|gif|png|ico|eot|svg|ttf|woff|woff2)$ {
  add_header Cache-Control "max-age=31536000, public";
}

```

This configuration file contains some optimizations for our Angular application. It will do the following:
- Redirect all paths to index.html, from this point on Angular will handle the routing - if we don't do this, we will get a 404 error when we try to access a route that is not the root route
- Gzip optimization
- Cache optimization


Now we create the `Dockerfile` in the root of the project and paste the following content into it:

```Dockerfile
# This is the intermediate image that we will use to build the bundle of the Angular application
FROM node:20 AS intermediate

WORKDIR /root

# Copy the package.json and package-lock.json files to the image and install the dependencies
COPY package.json package-lock.json ./
RUN npm i -g @angular/cli
RUN npm ci

# Copy the rest of the project into the image
COPY . .

# This is the place where you would linting and formatting and run the unit and e2e tests
# RUN ng test
# RUN ng lint

# Create the bundle of the Angular application
RUN ng build --configuration=production

# Using NGXINX as the final image
FROM nginx:1.23

# Copy the configuration files into the image
# Change this path if you have a different config folder structure
COPY build/nginx/*.conf ${NGINX_DEFAULT_CONF_PATH}

# Only copy the bundle/dist folder from the intermediate image to the final image
# TODO Change the path to your own dist folder
COPY --from=intermediate /root/dist/my-angular-app /usr/share/nginx/html

EXPOSE 80
```

Make sure you update the file paths to your own project structure.

Now we can build the Docker image and run it:
```bash
docker build . -t my-angular-app
docker run -p 8080:80 my-angular-app
```

Now you should be able to access your Angular application on [http://localhost:8080](http://localhost:8080){:target="_blank"}

<img src="/assets/angular-nginx-docker/angular-nginx-docker-in-browswer.png" alt="Angular application running in Docker" title="Angular application running in Docker">

Congratulations! You have successfully run your Angular application in an NGINX Docker container. 🎉
