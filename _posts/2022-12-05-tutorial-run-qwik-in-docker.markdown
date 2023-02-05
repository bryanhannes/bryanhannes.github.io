---
layout: post
title:  "Tutorial: Running Qwik in a Docker container"
date:   2022-12-05 16:00:00 +0100
categories: Qwik
published: true
comments: true
categories: Qwik
cover: assets/running-qwik-in-docker/running-qwik-in-docker-cover.png
tags: [Qwik, Docker]
type: article
---

In this tutorial, we are going to set up a new Qwik project, build a Docker image from our Qwik project and run the Docker image on our local machine.

At the time of writing the latest version of Qwik is 0.15.0 and Qwik City 0.0.127.

Let's assume that you have NPM, Node and Docker installed already.

## Generate Qwik project
We start with generating a new Qwik project.
We can follow the steps in the <a href="https://qwik.builder.io/docs/getting-started/" target="_blank">Qwik getting started guide</a> or follow the steps below.

Open a terminal and run the following command

```shell
npm create qwik@latest
```
The CLI will prompt for an application name, we are going to name our application: `qwik-docker`

Select the  `Basic App (QwikCity)` starter and install the npm dependencies.

The terminal should look like this:
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
npm start
```

Open a browser and go to <a href="http://localhost:5174" target="_blank">http://localhost:5174</a>

<img src="/assets/running-qwik-in-docker/new-qwik-project.jpg" alt="Newly generated Qwik project" title="Newly generated Qwik project">

## Adding the Express adaptor
The next step is to add the Express adaptor.

Run the following command in the terminal and select `Yes looks good, finish update!`

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

When Express is successfully installed we should see a success message in the terminal and 2 files should be created:
- `adaptors/express/vite.config.ts`
- `src/entry.express.tsx`

```shell
✔ Updating app and installing dependencies...
🦄  Success!  Added express to your app
```



The `entry.express.tsx` file is going to be the startup file of the application.

<img src="/assets/running-qwik-in-docker/results-after-express-installed.jpg" alt="Generated Express Adaptor folder structure" title="Generated Express Adaptor folder structure">


## Creating the dockerfile

Add a new Dockerfile to the root of the Qwik project with the following content.

```Dockerfile
# Intermediate docker image to build the bundle in and install dependencies
FROM node:19.2-alpine3.15 as build

# Set the working directory to /usr/src/app
WORKDIR /usr/src/app

# Copy the package.json and package-lock.json over in the intermedate "build" image
COPY ./package.json ./
COPY ./package-lock.json ./

# Install the dependencies
# Clean install because we want to install the exact versions
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
The moment we are waiting for is finally here, we are going to build the Docker image.

- Make sure the Docker daemon is running
- Open the terminal
- Navigate to the root directory of the Qwik project
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

When the build completes we can run the Docker image.

```shell
docker run -p 3000:3000 qwik-docker
```

Now we should be able to access the Qwik application which is running in Docker at
<a href="http://localhost:3000" target="_blank">http://localhost:3000</a>

<img src="/assets/running-qwik-in-docker/qwik-running-in-docker.jpg" alt="Qwik running on Docker" title="Qwik running on Docker">


## Conclusion
Congratulations, we just ran a newly created Qwik project in Docker
Now that we have our Docker image we can easily deploy it to the cloud with eg. Google Cloud Run.

## Note from author
I hope you enjoyed this blog article and learned a thing or two. Feel free to leave any remarks/improvements/feedback in the comments or hit me up on <a href="https://twitter.com/BryanHannes" target="_blank">Twitter</a>.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Feeling proud 💪<br>I finished my first blog post ever today: Tutorial: Running <a href="https://twitter.com/QwikDev?ref_src=twsrc%5Etfw">@QwikDev</a> in a <a href="https://twitter.com/Docker?ref_src=twsrc%5Etfw">@Docker</a> container&quot;<br><br>Find the blog article here:<a href="https://t.co/RyoYxAfr5G">https://t.co/RyoYxAfr5G</a><br><br>Hope you enjoy it and feedback is always welcome 😍</p>&mdash; Bryan Hannes (@BryanHannes) <a href="https://twitter.com/BryanHannes/status/1599790550491987969?ref_src=twsrc%5Etfw">December 5, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## Video

<iframe width="100%" height="500" src="https://www.youtube.com/embed/knKeJDGTNpk" title="Tutorial: Running Qwik in a Docker container" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Reviewers
Special thanks to the reviewer: <a href="https://twitter.com/brechtbilliet" target="_blank">Brecht Billiet</a> from <a href="https://simplified.courses/" target="_blank">Simplified Courses</a>

