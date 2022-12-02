---
layout: post
title:  "Running Qwik in a Docker container"
date:   2022-12-01 00:00:00 +0100
categories: Qwik
---

# Running Qwik in a Docker container
In this article, I will show you how you can quickly set up a new Qwik project and run it in a Docker container.

At the time of writing the latest version of Qwik is 0.15.0 and Qwik City 0.0.127.

We assume that you have NPM, Node and Docker installed already.

## Generate Qwik project
We start with generating a new Qwik project. 
You can follow the steps in the <a href="https://qwik.builder.io/docs/getting-started/" target="_blank">Qwik getting started guide</a> or follow the steps in this below.

Open your terminal

```shell
> npm create qwik@latest
```

Give your application a name, we are going to name our application: *qwik-docker*

Select a starter, for this article we are going to create a *Basic App (QwikCity)"*

And we will install the npm dependencies already.

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
We are going to navigate to the newly created project and start the local development environment

```shell
> cd qwik-docker
> npm run start
```

Open your browser and go to [http://localhost:5174/](http://localhost:5174/). 

<img src="/assets/running-qwik-in-docker/new-qwik-project.jpg" alt="Newly generated Qwik project" title="Newly generated Qwik project">

## Adding the Express adaptor
We have to add an Express adaptor to be able to run Qwik in Docker. 

Run following command in the terminal and select "Yes looks good, finish update!"

```shell
> npm run qwik add express

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

When Express is succesfully installed you should see:
```shell
✔ Updating app and installing dependencies...
🦄  Success!  Added express to your app
```

The command should have created a adaptors/express/vite.config.ts and a src/entry.express.tsx file.

The entry.express.tsx file is going to be the startpoint of the application.

<img src="/assets/running-qwik-in-docker/result-after-express-installed.png" alt="Generated Express Adaptor folder structure" title="Generated Express Adaptor folder structure">


## Creating the dockerfile

Add a new Dockerfile to the root of your project with the following content.

// TODO add comments to the dockerfile to explain it more
// Add the Dockerfile to a github repo
```Dockerfile
FROM node:18.12.1-alpine3.15 as build

WORKDIR /usr/src/app

COPY ./package.json ./
COPY ./package-lock.json ./

RUN npm ci

COPY ./ ./

RUN npx npm run build

FROM node:18.12.1-alpine3.15 as production

WORKDIR /usr/src/app

COPY ./package.json ./
COPY ./package-lock.json ./

RUN npm ci

COPY --from=build /usr/src/app/server ./server
COPY --from=build /usr/src/app/dist ./dist

EXPOSE 3000

CMD [ "node", "server/entry.express"]
```

## Build and run project in Docker

Now we are going to build the Docker image

Open the terminal and navigate to the root directory of your project
```shell
> docker build -t qwik-docker .
```
The build should succeed fairly quickly:
```shell
~/qwik-docker ”bryan”⬢ v16.14.2 took 1m 19s 
➜  docker build -t qwik-docker .
[+] Building 37.6s (14/14) FINISHED                                                                                                                     
 => [internal] load build definition from Dockerfile                                                                                               0.0s
 => => transferring dockerfile: 476B                                                                                                               0.0s
 => [internal] load .dockerignore                                                                                                                  0.0s
 => => transferring context: 2B                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/node:18.12.1-alpine3.15                                                                         1.3s
 => [build 1/7] FROM docker.io/library/node:18.12.1-alpine3.15@sha256:cd3a7004267e419477bbfc50e0502df8607a0b9b4465092f6e2c2ce4092faa45             0.0s
 => [internal] load build context                                                                                                                  4.3s
 => => transferring context: 176.67MB                                                                                                              4.2s
 => CACHED [build 2/7] WORKDIR /usr/src/app                                                                                                        0.0s
 => [build 3/7] COPY ./package.json ./                                                                                                             0.8s
 => [build 4/7] COPY ./package-lock.json ./                                                                                                        0.1s
 => [build 5/7] RUN npm ci                                                                                                                        10.9s
 => [build 6/7] COPY ./ ./                                                                                                                         2.6s
 => [build 7/7] RUN npx npm run build                                                                                                             15.0s 
 => [production 6/7] COPY --from=build /usr/src/app/server ./server                                                                                0.0s 
 => [production 7/7] COPY --from=build /usr/src/app/dist ./dist                                                                                    0.0s 
 => exporting to image                                                                                                                             2.1s 
 => => exporting layers                                                                                                                            2.1s 
 => => writing image sha256:11c6b56b8278d4503cf989b9a0680da0a5cbc05f9800493de1a5ddf3da81c79f                                                       0.0s 
 => => naming to docker.io/library/qwik-docker                                                                                                     0.0s 
```

When the build completes you can run the newly created image with:

```shell
> docker run -p 3000:3000 qwik-docker
```

Now you should be able to access your Qwik application which is running Docker at 
<a href="http://localhost:3000" target="_blank">http://localhost:3000</a>

<img src="/assets/running-qwik-in-docker/qwik-running-in-docker.jpg" alt="Qwik running on Docker" title="Qwik running on Docker">


# Conclusion
Congratulations you just managed to run a newly created Qwik project in Docker
Next steps
- now you can easily deploy your Qwik application to the cloud. Eg Google Cloud Run 
