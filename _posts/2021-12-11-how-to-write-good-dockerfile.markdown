---
title: How to write a good Dockerfile?
layout: post
date: 2021-12-11 00:00:00 +0200
categories: docker
---

# What is Docker?

Docker provides tools to package and run your application in what is called a container. You can think of a container like a self-contained package, that is more lightweight than a regular virtual machine. A container contains all the necessary code and dependencies to run your application, but it doesn’t contain (or need) the operating system inside the container. Instead, Docker containers running on the host share the operating system’s kernel share the host’s resources, and they run on the host’s userspace as isolated processes. Because the containers consume much fewer resources on the host than regular virtual machines, you can run more containers on a single host and use the host’s resources more efficiently compared to regular virtual machines. The downside is that docker containers offer less isolation compared to virtual machines, as the containers share the host’s resources, instead of that containers would have some resources directly allocated to them.

## How do I use docker in my projects?

So containers are neat, but how can you benefit from them in your day-to-day work? I use containers in two ways. When I’m writing software, docker is an exceptionally good tool to run all the dependencies that your application needs. For example, if I’m working on a web application, that application usually needs some sort of a database to save data into. Docker addresses this issue with a tool called Compose. I won’t deal Compose now, but instead, I’ll write about Compose another time.

The other way I use Docker in my day-to-day work, and where I’m going to focus on this post, is packaging my application in a neat container that can be run the same way as it would be run if it is deployed into any service capable of running containers (like Azure AppService). The power of containers for me is that I can easily create the exact same running environment on my own computer as when the application is deployed into production. This creates more confidence when writing code, that the code you write, will actually run the same way in production. When I spin up the container locally, I can verify that the application starts correctly and that it works as I expect.

We can also use Docker to run our unit test suites. So you can run your tests in the same environment they will be run in your continuous integration (CI) server. That is a huge benefit and something that I highly recommend you to try if you have not done that yet.

One last benefit that Docker brings to the table is that by containerizing your application you can make new developers join your project easier. When you can spin up your application locally by just running docker run myapp, it’s a huge quality of life improvement over going through all the documentation and manually installing all the required dependencies. And this also works great when you need to hop between multiple projects. It’s much easier to spin up a docker container(s) than manually starting services.

## What is a Dockerfile?

### Simple Dockerfile

You have probably seen many Dockerfiles before, but in case you aren’t or just need a refresher, here’s how Dockerfiles look like.

```Dockerfile
FROM node:latest
USER node
RUN echo 'console.log("hello world!")' > index.js
CMD ["node", "index.js"]
```

In this example. I’m building my application on top of the latest version of the official node.js image. I’m also issuing one command to echo some contents into index.js file, and finally, I’m telling docker to run that index.js file when the container is started.

I can build this image with (addressing it a tag simple:latest):

```s
$ docker build -t simple:latest .
```

And when I run the container I can see that it will run the index.js file.

```s
$ docker run -it simple:latest
hello world!
```

## Recipe of a lean Dockerfile

When writing Dockerfiles, you want to aim for the smallest image possible. The smaller the image, the faster it is to upload to a repository and (obviously) the less space it requires. Another thing you want to optimize for is building the image as quickly as possible. The less time it takes to build the image, the easier it is to use in your local environment, and the faster it makes your build process.

There are couple of things you need to remember that will help you build better dockerfiles.

### Alpine images

If your application can be built and run on Alpine-based images (musl libc, and busybox) I recommend you to do so. Alpine-based images are usually multiple times smaller than GNU Linux-based images so the final image is also smaller. Alpine provides a very good package manager, so it is almost always possible to tailor the container for your needs. Sometimes, though, there might be a case when your application needs something that doesn’t work on alpine, and those times you need to revert to something else like ubuntu. But I always recommend starting with alpine and seeing if it suits your situation. That way your base image adds as little weight to your final image as possible.

### Multi-stage builds

We can cut some additional weight by using multi-stage builds. This means that you can split your image build process into parts, and then in the final build stage you just pick the pieces you want to include in your final image. I tend to think this so that I first want to install all the required dependencies into the image, then I compile my source files, and in the last stage I put all the pieces together to produce the final image. Because with multi-stage builds, only the last stage is included into the final image, this leads to images with fewer layers. Using this method you can also build images that contains only the necessary files to run your production application. So all the build time requirements can be discarded.

This is how multi-stage build can look like when we build a typescript app

```Dockerfile
FROM node:alpine AS dev-deps
USER node
COPY --chown node:node package.json package-lock.json ./
RUN npm install

FROM node:alpine AS production-deps
USER node
COPY --chown node:node package.json package-lock.json ./
RUN npm install --production

FROM dev-deps AS compile # we need devDependencies to run tsc
USER node
COPY --chown node:node . ./
RUN tsc . # We can run typescript compiler

FROM node:alpine AS final
USER node
COPY --chown node:node --from-stage=production-deps /node_modules ./node_modules
COPY --chown node:node --from-stage=compile /dist ./dist
CMD ["node", "dist/index.js"]
```

In this example, you can see that each FROM instruction has an alias set for it with AS instruction. Using AS you can refer to that stage in later parts of the Dockerfile. In the example above we are continuing from the dev-deps stage in the compile stage, so we can use the devDependencies installed in the previous stage when we compile typescript sources with tsc.

So is this all worth it? In short, yes. With multi-stage builds, you can make your final images leaner. That is possible because you can define the final stage so that you only copy necessary files to the final image, so you can skip all those intermediate layers that you need in the previous stages to build your application.

If we were to build this app without a multi-stage build, the Dockerfile could look something like this

```Dockerfile
FROM node:alpine AS deps
USER node
COPY --chown node:node package.json package-lock.json ./
RUN npm install
COPY --chown node:node . ./
RUN tsc . # Run typescript compiler
RUN rm -rf node_modules/
RUN npm install --production
CMD ["node", "dist/index.js"]
```

This image would contain 9 layers compared to 5 layers in the multi-stage image (final stage). But the bigger problem with this one-stage build is that now the final image still contains the original typescript files (and any other files we copied into the container and what we didn’t clean up manually) in the container even though those are not needed. So multi-stage builds let you build leaner images with less effort because you can always start the last stage from scratch and copy only the necessary files to the final image.

Multi-stage images have other benefits as well. You can use those stage aliases as build targets. Let’s look at that next.

## Using build targets to run tests

One common case I use the build targets for is that I’ll want to run my project’s unit tests with docker. Let’s see how that can be achieved by using the example from above:

```Dockerfile
FROM node:alpine AS dev-deps
USER node
COPY --chown node:node package.json package-lock.json ./
RUN npm install

FROM node:alpine AS production-deps
USER node
COPY --chown node:node package.json package-lock.json ./
RUN npm install --production

FROM dev-deps AS compile # we need devDependencies to run tsc
USER node
COPY --chown node:node . ./
RUN tsc . # We can run typescript compiler

FROM node:alpine AS final
USER node
COPY --chown node:node --from-stage=production-deps /node_modules ./node_modules
COPY --chown node:node --from-stage=compile /dist ./dist
CMD ["node", "dist/index.js"]

FROM compile AS test
CMD ["npm", "test"]
```

Now this image contains an additional test stage that can be used to run your application’s unit test suite. If you build this image with docker build . —tag test then you can run the tests as docker run -it test.

Now you may ask, that how can we now use this image in production because it contains a test stage in the end? Well turns out that you can use docker build with —target flag to build up to a specific target. So running `docker build . —tag production:latest —target=final` would build an image ready for production. This image won’t contain the last test stage, but the final image is what is defined in the final stage.

If this feels a bit clunky way to run your tests, you can ease the burden by scripting those commands. So you can leverage npm scripts for example:

```json
#package.json
...
scripts: {
    "docker:build": "docker build . --tag app:latest --target=final",
    "docker:test": "docker build . -t test && docker run -it test"
}
```

Now you can easily build your app by running npm run docker:build, and run the tests by npm run docker:test.

This way of running your tests is really nice because now you have a way to run the tests exactly the same way they are run on your Continuous Integration environment. This reduces the probability that you will encounter tests that work on your machine but breaks when run on CI. And I think that is really valuable.

Structure your Dockerfile to achieve a high level of caching
One last thing I want to tell you about is how you can achieve a Dockerfile that rebuilds fast, so you can use that while developing your application locally. If Dockerfile is slow to build and clunky to use, there is a high probability that you won’t use it long in your local environment. And that would be a real shame. So let’s see what we can do to make our image build quickly.

One powerful feature of Docker is that it has a built-in caching, that it utilizes as much as it can. When you build your Dockerfile, docker then executes each instruction in that Dockerfile, and it will create a layer from the result of each instruction. Docker caches these layers so it can use them the next time you build the same Dockerfile. When you build the same Dockerfile again, Docker will try to use the layers from its cache if possible. If Docker cannot use the layer from the cache, then it needs to execute the instruction again. For example, if you copy some files into the image, you want docker to copy the changed files into the image the next time you build. But if no source files have changed, you would rather want docker to use the cached version from the previous run, so no unnecessary work is done.

Let’s compare two Dockerfiles:

# Dockerfile A

```Dockerfile
FROM node:latest
USER node

COPY . ./
RUN npm install

CMD ["node", "index.js"]
```

So what is the issue here? The first build works just fine. If we then change some source files and build this image again, we see that npm install will be run again even though we didn’t update any dependencies. Why is that? That happens because Docker sees that COPY . ./ results in a different layer than in the previous run, so docker needs to invalidate the rest of the cached layers in this Dockerfile. That’s why it needs to execute the RUN npm install again.

We can fix this by installing the dependencies before we copy any source files into the container:

# Dockerfile B

```Dockerfile
FROM node:latest
USER node

COPY package.json ./
RUN npm install
COPY . ./

CMD ["node", "index.js"]
```

Now dependencies are only installed again if you change something inside the package.json file.

This is a simplified view of how docker caching works. You should follow the same principle with everything else when building your Dockerfiles. Handle all the frequent changes last, so you can benefit from the already cached layers as much as possible.

Final words
In this post, we have gone through some of the basics of Docker to see how we can create a Dockerfile that is good to use both locally and in your production setup. I didn’t go through everything you need to take care of to build production-ready images, but what I described here should give you a rough idea of where to aim at. If you are interested in a checklist of what needs to be done to create a production-ready, solid nodejs docker container, then I highly recommend checking out this Snyk’s article.

And if you are interested in an excellent ready-to-use typescript node docker container that already checks all of the boxes of that above article, you should check out the dockerfile I created just for that.

And finally, I would love to hear what you think about this article.

References:

https://en.wikipedia.org/wiki/Docker_(software)
https://www.docker.com/resources/what-container
