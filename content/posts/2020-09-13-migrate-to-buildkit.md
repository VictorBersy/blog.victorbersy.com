---
title: "Improve Docker build time by using BuildKit and SemaphoreCI"
date: 2020-09-13T13:00:09+02:00
draft: false
---

## Context

At Selectra, we use Docker on all our environments: from local to production, and for our CI pipelines.

The first CI step is to build a Docker image that will be used during the whole pipeline, and eventually deployed if tests pass.

[![Our CI pipeline](/2020-09-13-migrate-to-buildkit/ci-pipeline.png)](/2020-09-13-migrate-to-buildkit/ci-pipeline.png)

The faster the build, the faster the tests and others things developers cares about will start, so it's crucial to not "waste" minutes on builds.

We mostly have web projects, and they all have some kind of dependencies:
- PHP packages through `composer`
- JS packages through `npm`
- PHP extensions
- etc...

All of these dependencies can require a lot of time to be installed. And so, in order to keep the build time as short as possible, caching their installation could make our build much shorter.

While most CI providers has tools to cache theses dependencies in a way or another, ideally, you'd like to cache Docker layers to make things easier. You won't have to setup different tools for different packages providers, and you can even cache others things like `apt-get`, `apk`, etc... that you might run during your build.

Here's a typical mutli-stage Dockerfile that we use for our PHP applications:
```dockerfile
FROM composer:2 as composer-build
...
FROM node:10-alpine as npm-build
...
FROM php:7.4-apache as php-extensions-build
...
# Final build, "app", our base for others
FROM php:7.4-apache as app
...
# Various stages for release/dev/etc...
FROM app as release
...
# Back dev stage, install backend dev tools here
FROM app as dev-back
...
# Front dev stage, install frontend dev tools here
FROM npm-build as dev-front
...
# Test stage, used during CI/CD and local testing
FROM app as test
...
# Default docker build target will be 'app'
FROM app
```

We use multistage Docker build so we can pick exactly what we want for release images, or for our dev environments, keeping our images as light as we can.

## Issues we were facing

### Caching all docker stages manually is hard

The easiest solution at first sight would be to build each stage separately and tag them:
```bash
docker build -t my-project:composer-build --target=composer-build .
docker build -t my-project:npm-build --target=npm-build .
docker build -t my-project:php-extensions-build --target=php-extensions-build .
# etc...
```

And send everything to a container registry such as AWS ECR:
```bash
docker push aws_account_id.dkr.ecr.region.amazonaws.com/my-project:composer-build
docker push aws_account_id.dkr.ecr.region.amazonaws.com/my-project:npm-build
docker push aws_account_id.dkr.ecr.region.amazonaws.com/my-project:php-extensions-build
# etc...
```

Then you'd have to pull each of these stages before starting to build your image.

But you may have noticed it's now a lot of things to manually setup, and maintain if you add a new stage at some point.

### Running builds in parallel is hard

For example, `composer-build` and `npm-build` does not depend on each other, so you could build them in parallel.

But you would have to write your own parser, or just come up with some bash scripts to do that, and maintain it over time too. That would be tedious! Specially when you know this dependency graph is already described in your Dockerfile.

Maybe I'm missing something, but as far as I known, without BuildKit, you can't easily build stages in parallel.

### AWS ECR or any other remote registry are too slow to cache our stages

Another issue that might depend on your CI provider: pushing Docker images to remote registry (AWS ECR, GCP Container Registry, etc...) is taking a long time.

Pulling/pushing each stage to a remote registry is going to take a significant amount of your build time step. Plus, it would clutter your registry with intermediate stages that you don't care about long-term wise.

## Solution: Using BuildKit

To solve our issues related to caching and parallel builds, we are now using BuildKit.

Thanks to [BuildKit project](https://github.com/moby/buildkit), we can easily solve all our issues with a single command:

```bash
buildctl build ... \
  --output type=image,name=localhost:5000/myrepo:image-test,push=true \
  --export-cache type=registry,ref=localhost:5000/myrepo:buildcache,push=true \
  --import-cache type=registry,ref=localhost:5000/myrepo:buildcache \
  --opt target=test
```

As we are asking to target the `test` stage, we are also sure to build only what we want without having to maintain any kind of manual scripts.

What we are saying to BuildKit is:
- The final image created must be pushed to `localhost:5000/myrepo:image-test`
- Export/Import cache to/from `localhost:5000/myrepo:buildcache`
- We want to build the test image, so we pass `--opt target=test` as a parameter.
  - This way all our tests tools will be there for next CI steps.
  - To create the release image, just change it to `--opt target=release` and name the new image as you'd like.

## Solution: Use your CI provider private Docker registry

I'm not sure if others CI providers has something similar, but SemaphoreCI had the good idea to deploy their own Docker registry, where it's much faster to push your cache and images.

The push time is excellent, much better than on AWS ECR.

Thanks to this, we can use their private Docker registry to push/pull our intermediate stages and cache layers.

## Before / after + screenshots

On a basic Laravel applications, here's the results.

**Before BuildKit:**

[![Before using BuildKit](/2020-09-13-migrate-to-buildkit/laravel-before.png)](/2020-09-13-migrate-to-buildkit/laravel-before.png)

**After BuildKit:**

[![After using BuildKit](/2020-09-13-migrate-to-buildkit/laravel-after.png)](/2020-09-13-migrate-to-buildkit/laravel-after.png)

Before using BuildKit, the pipeline was taking around ~3min30s. It's now taking ~2min. Though, it depends on what cache layers needs to be rebuild. If none of them are changing, like if you are running again the same build and it's already cached, it takes ~14s.

## What improvements I'd like to make?

### Tracing our CI pipelines

I'm a fan of observability and I want to apply this methodology on our CI/CD pipelines too. BuildKit comes with a way to export traces to Jaeger, and it can be visualized like so:

[![BuildKit traces exported and visualized with Jaeger](/2020-09-13-migrate-to-buildkit/buildkit-traces.jpg)](/2020-09-13-migrate-to-buildkit/buildkit-traces.jpg)

By doing so, we could track over time the build time easily for each step. We could see what impact our changes does have and on what steps. We can also avoid wasting time optimizing a step if there is another that takes 90% of the build time.

### Move out from CI the push/tag release steps

You may have noticed that I am building, pushing and tagging the release candidate image during the CI pipeline. I did not realized while upgrading our CI/CD pipelines that it does not make sense to have these things there.

Ideally, I'd like to have something like:
* **CI:**
  * Build test image
  * Run tests
  * Build release candidate image
* **CD:**
  * Push release candidate image to AWS ECR
  * Tag release candidate as "release" on AWS ECR
  * **Deploy to X**
    * Deploy to Kubernetes development
    * Deploy to Kubernetes staging
    * Deploy to Kubernetes production

By doing so, I could make the CI pipeline even shorter and our developers would received automated feedbacks faster.

That would give something like:

[![BuildKit traces exported and visualized with Jaeger](/2020-09-13-migrate-to-buildkit/moving-out-docker-push-to-cd.png)](/2020-09-13-migrate-to-buildkit/moving-out-docker-push-to-cd.png)
