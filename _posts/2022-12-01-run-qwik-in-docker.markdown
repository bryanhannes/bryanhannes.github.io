---
layout: post
title:  "Running Qwik in a Docker container"
date:   2022-12-01 00:00:00 +0100
categories: Qwik
published: true
comments: true
categories: Qwik
cover: assets/running-qwik-in-docker/running-qwik-in-docker-cover.png

---

Last week I started learning Qwik, I must say I see a lot of potential in this framework. 
As an experiment I wanted to run a Qwik project in a Docker container. In this article, I will show you how I did it.

At the time of writing the latest version of Qwik is 0.15.0 and Qwik City 0.0.127.

I assume that you have NPM, Node and Docker installed already.

## Generate Qwik project
We start with generating a new Qwik project. 
You can follow the steps in the <a href="https://qwik.builder.io/docs/getting-started/" target="_blank">Qwik getting started guide</a> or follow the steps in this below.

Open your terminal and run following command

```shell
npm create qwik@latest
```
The CLI will prompt for an application name, we are going to name our application: *qwik-docker*

Select the  *Basic App (QwikCity)* starter and install the npm dependencies.

Your terminal should look like this:
```shell
🐰 Let's create a Qwik app 🐇   v0.15.0

✔ Where would you like to create your new project? … qwik-docker

✔ Select a starter › Basic App (QwikCity)

✔ Would you like to install npm dependencies? … yes

✔ Installing npm dependencies...


🦄  Success!  Project created in qwik-docker directory

🐰 Next steps:
   cd qwik-docker
   npm start

🔌 Integrations? Add Netlify, Cloudflare, Tailwind...
   npm run qwik add

📚 Relevant docs:
   https://qwik.builder.io/docs/getting-started/

💬 Questions? Start the conversation at:
   https://qwik.builder.io/chat
   https://twitter.com/QwikDev

📺 Presentations, Podcasts and Videos:
   https://qwik.builder.io/media/
```

## Run the newly generate Qwik project locally
Now we are going to check if the new Qwik project runs locally.
We are going to navigate to the newly created project and start the local development environment.

```shell
cd qwik-docker
npm run start
```

Open your browser and go to <a href="http://localhost:5174" target="_blank">http://localhost:5174</a>

<img src="/assets/running-qwik-in-docker/new-qwik-project.jpg" alt="Newly generated Qwik project" title="Newly generated Qwik project">

## Adding the Express adaptor
The next step is to add the Express adaptor.

Run following command in the terminal and select *"Yes looks good, finish update!"*

```shell
npm run qwik add express
```

```shell

👻  Ready?  Add express to your app?

🐬 Modify
   - README.md
   - package.json

🌟 Create
   - src/entry.express.tsx
   - adaptors/express/vite.config.ts

💾 Install npm dependencies:
   - @types/compression ^1.7.2
   - @types/express 4.17.13
   - compression ^1.7.4
   - express 4.17.3

? Ready to apply the express updates to your app? ›  
❯   Yes looks good, finish update!
    Nope, cancel update

```

When Express is succesfully installed you should see a success message in the terminal:
```shell
✔ Updating app and installing dependencies...
🦄  Success!  Added express to your app
```

and 2 files should be created: 
- adaptors/express/vite.config.ts
- src/entry.express.tsx

The *entry.express.tsx* file is going to be the startup file of the application.

<img src="/assets/running-qwik-in-docker/result-after-express-installed.png" alt="Generated Express Adaptor folder structure" title="Generated Express Adaptor folder structure">


## Creating the dockerfile

Add a new Dockerfile to the root of your project with the following content.

```Dockerfile
# Intermediate docker image to build the bundle in and install dependencies
FROM node:19.2-alpine3.15 as build

# Set the working directory to /usr/src/app
WORKDIR /usr/src/app

COPY ./package.json ./
COPY ./package-lock.json ./

# Install the dependencies
RUN npm ci

# Copy the source code into the build image
COPY ./ ./

# Build the project
RUN npm run build

# Pull the same Node image and use it as the final (production image)
FROM node:19.2-alpine3.15 as production

# Set the working directory to /usr/src/app
WORKDIR /usr/src/app

# Only copy the results from the build over to the final image
# We do this to keep the final image as small as possible
COPY --from=build /usr/src/app/node_modules ./node_modules
COPY --from=build /usr/src/app/server ./server
COPY --from=build /usr/src/app/dist ./dist

# Expose port 3000 (default port)
EXPOSE 3000

# Start the application
CMD [ "node", "server/entry.express"]
```

## Building and running the project in Docker
The moment you are waiting for is finally here: we are going to build the Docker image.

- Make sure the Docker daemon is running
- Open the terminal 
- Navigate to the root directory of your Qwik project 
- Run the build command

```shell
docker build -t qwik-docker .
```

```shell
~/demos/qwik-docker ”bryan”⬢ v16.14.2 on 🐳 v20.10.14 
➜  docker build -t qwik-docker .                                                                                                               
[+] Building 16.8s (15/15) FINISHED                                                                                                                                                                                     
 => [internal] load build definition from Dockerfile                                                                                                                                                               0.0s
 => => transferring dockerfile: 827B                                                                                                                                                                               0.0s
 => [internal] load .dockerignore                                                                                                                                                                                  0.0s
 => => transferring context: 2B                                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/node:19.2-alpine3.15                                                                                                                                            0.5s
 => [internal] load build context                                                                                                                                                                                  0.5s
 => => transferring context: 510.88kB                                                                                                                                                                              0.4s
 => [build 1/7] FROM docker.io/library/node:19.2-alpine3.15@sha256:12d9c7253f232bb88a9ef6d6e974afd90e296cb8383572dbb7f28c39f828b07e                                                                                0.0s
 => CACHED [build 2/7] WORKDIR /usr/src/app                                                                                                                                                                        0.0s
 => CACHED [build 3/7] COPY ./package.json ./                                                                                                                                                                      0.0s
 => CACHED [build 4/7] COPY ./package-lock.json ./                                                                                                                                                                 0.0s
 => CACHED [build 5/7] RUN npm ci                                                                                                                                                                                  0.0s
 => [build 6/7] COPY ./ ./                                                                                                                                                                                         1.3s
 => [build 7/7] RUN npm run build                                                                                                                                                                                 13.6s
 => CACHED [production 3/5] COPY --from=build /usr/src/app/node_modules ./node_modules                                                                                                                             0.0s
 => CACHED [production 4/5] COPY --from=build /usr/src/app/server ./server                                                                                                                                         0.0s
 => [production 5/5] COPY --from=build /usr/src/app/dist ./dist                                                                                                                                                    0.0s
 => exporting to image                                                                                                                                                                                             0.0s
 => => exporting layers                                                                                                                                                                                            0.0s
 => => writing image sha256:6a3a1f7e57b6974f2552208ba28e9e9487141e157f3f3ecd32a20372a4bf6341                                                                                                                       0.0s
 => => naming to docker.io/library/qwik-docker                                                                                                                                                                     0.0s                                                                   0.0s 
```

When the build completes you can start the image.

```shell
docker run -p 3000:3000 qwik-docker
```

Now you should be able to access your Qwik application which is running in Docker at 
<a href="http://localhost:3000" target="_blank">http://localhost:3000</a>

<img src="/assets/running-qwik-in-docker/qwik-running-in-docker.jpg" alt="Qwik running on Docker" title="Qwik running on Docker">


# Conclusion
Congratulations, you just ran a newly created Qwik project in Docker
Now that you have your image you can easily deploy it to the cloud with eg. Google Cloud Ru

# Note from author
I hope you enjoyed this blog article and learned a thing or two. Should you have any remarks/improvements/feedback, please hit me up via <a href="https://twitter.com/BryanHannes" target="_blank">Twitter</a>.
