---
layout: post
title:  "Running Qwik in Docker without Dockerfile (Buildpacks)"
date:   2022-12-24 01:00:00 +0100
published: true
comments: true
categories: Qwik
cover: assets/running-qwik-in-docker-without-dockerfile-buildpack/running-qwik-in-docker-without-dockerfile-buildpack.png
tags: [Qwik]
type: article
---

In this tutorial, we are going to create an OCI image (Docker image) from a Qwik application without providing a Dockerfile. We can do this with a tool called [Buildpacks](https://buildpacks.io/){:target="_blank"}.
In a previous article, we already discussed how to build and run [Qwik in Docker](/tutorial-run-qwik-in-docker){:target="_blank"}.

At the time of writing the latest version of Qwik is 0.16.1 and Qwik City 0.0.128.

Let's assume that you have NPM, Node and Docker already installed.

## Setting up a Qwik project
We are going to set up a new Qwik project and add the Express adaptor.
To do this we follow the steps "Generate Qwik project" and "Adding the Express adaptor" from a [previous tutorial](/tutorial-run-qwik-in-docker){:target="_blank"}.

We named our Qwik application `qwik-buildpack-app`.

## Building the OCI image
Now that we have our Qwik application we can start building the OCI image with `Buildpack`.

### Installing Pack CLI
First, install the Pack CLI tool. We will follow the [Buildpack documentation](https://buildpacks.io/docs/tools/pack/#pack-cli){:target="_blank"} for this.

Verify that Pack is successfully installed.
```shell
pack version
```

### Setting the default builder

We are going to set up `heroku/buildpacks` as our default builder for this tutorial. There are other builders as well that you can use, some well-known builders are [Paketo Buildpacks](https://paketo.io/){:target="_blank"} and Google Cloud Buildpacks.

To use the `Pack CLI`, the Docker daemon should be running.
We can verify that the Docker daemon is running with:
```shell
docker version
```

Then set the default builder to `heroku/buildpacks`.
```shell
pack config default-builder heroku/buildpacks
```

### Adding a Procfile
A [Procfile](https://devcenter.heroku.com/articles/procfile){:target="_blank"} is a simple text file without extension that specifies commands that are run when the container starts up. 
This `Procfile` should always be placed in the root of the application.

The default startup point when creating a NodeJS OCI image with Buildpack is `npm start` but sometimes you don't want this.
We can leverage the Procfile to override the startup behavior of the container.

Open the root of the Qwik application in a terminal.

We can add the `Procfile` via the CLI or we can do this manually. 

```shell
 echo "web: node server/entry.express" > Procfile
```


### Building the application with Pack

Now we are going to build the OCI image with `Pack`.
Make sure that your terminal is opened at the root of the Qwik project.

```shell
pack build qwik-buildpack-app
```

### Running the Qwik app with Docker

When the build completes we can run the OCI image with  `Docker`.

```shell
docker run -p 3000:3000 qwik-buildpack-app
```

Open a browser on [http://localhost:3000](http://localhost:3000){:target="_blank"} to see the Qwik application running.

## Conclusion
Congratulations, we just build an OCI image (Docker image) from a Qwik application without a Dockerfile. 

In my humble opinion, Buildpacks are ideal for simple projects when you don't have a Dockerfile at hand. 
In more complex situations it may be better to write Dockerfiles.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Having a great feeling after a productive morning 💪 <br>I wrote a blog article how to run <a href="https://twitter.com/QwikDev?ref_src=twsrc%5Etfw">@QwikDev</a> in Docker without creating a Dockerfile with the help of <a href="https://twitter.com/buildpacks_io?ref_src=twsrc%5Etfw">@buildpacks_io</a> <br><br>Feedback is always welcome 😃<a href="https://t.co/vrNMlenGZE">https://t.co/vrNMlenGZE</a></p>&mdash; Bryan Hannes (@BryanHannes) <a href="https://twitter.com/BryanHannes/status/1606561696369328131?ref_src=twsrc%5Etfw">December 24, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
