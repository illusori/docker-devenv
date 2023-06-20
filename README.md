# docker-devenv

A context-aware use of docker as a development environment for language-agnostic projects.

## Why?

I work with lots of different languages for different projects...

 * I don't want to have to remember the individual build commands for 18 different languages.
 * Or the quirks of whatever system they use to isolate package management from the OS packages (if it's even possible in that language).
 * Or how to even run the project.
 * Or how to invoke the test suite.
 * Or if there's a mechanism to watch for file edits and autorun the tests.
 * Or how to spin up the docker instances in production.
 * For that matter I don't want the process of how I run the project locally to widly differ from the CI test machine, or staging, or production, more differences means more bugs.
 * I don't want to have to rely on having the right software installed on my development machine, not just dependencies with in the language package manager but OS level package management. The same goes double for contributors.

docker-devenv aims to resolve these issues. You get a simple set of commnds like `docker-devenv build`, `docker-devenv test` and _whereever_ you run them, they figure out what needs doing and runs the correct command: it Just Does It. They're context-aware commands.

## How?

Broadly, you have:

 * the host machine: this is where you edit your source files.
 * the devenv container: this is a docker container where all the dev dependencies and toolchain you need for development are installed, this ensures complete isolation from your host toolchain. You'll run your dependency management tooling here and any of the toolchain. (The host doesn't care if you have npm installed or what version!)
 * the build container: this is a docker container that has everything needed to build your project.
 * the runtime container: this is a docker container that is used for running your project.

Depending on what task you need to achieve, it will need to run in the appropriate container. However, that doesn't mean that YOU the developer need to run the command on that container. Just run the command on the host and docker-devenv will route it to the appropriate place.

ie: if you run `docker-devenv run` on the host machine as development machine, it will first run the devenv container, and then within that devenv container it will run `docker-devenv run` again, which will start up the runtime container. In a non-development environment it will directly run the runtime container. The runtime container itself runs `docker-devenv run` and inside the context of the runtime container that will directly start up your project app.

(Note: the devenv container uses socket magic to avoid running Docker-in-Docker and the associated problems that has, so all containers run directly on the host, they're just orchestrated from the devenv container.)
(Another Note: the default is to bundle and use the `docker-devenv` script within the runtime container to run the app for consistency, but it's entirely within your control not to for those situations where you may wish to keep your image lean, or you simply don't have POSIX sh installed in the runtime. Everything will still work if you want to keep the runtime image as uncontaminated as possible.)

## Installing and configuration

1. Download and install `docker-devenv` in your project, I suggest creating a `bin` dir. Ensure it has execute permissions set.
1. Copy and modify the sample `env` script in the same directory as `docker-devenv`, this contains some minimal configuration like the project name and what account name to use in the docker image prefixes. It also has some environment detection functions defined there so that you can source them for your own plugins, but you can ignore that for now.
1. Create as a minimum a `custom-run` script in the same dir, that will be sourced to start up your project within the runtime container. Look in the examples folder for examples but this will usually be the command you'd use as the `CMD` entry in your Dockerfile. This will run in your runtime container.
1. Optionally create a `custom-test` script which... invokes your test suite. This will run in your devenv container.
1. Optionally create a `custom-build` script which does any build steps that aren't part of your Dockerfile build stage for whatever reason. This will run in your build container.
1. The hardest part: create a multistage Dockerfile following the structure in the examples dir, but with your own needs substituted in. You minimally need a build stage, a devenv stage and a runtime stage. They should have their `CMD` kept to the invocations of `docker-devenv` provided in the example to start with.

That's it, you're now good to go.

## How to use

Edit your files on your host using your usual IDE/editor.

Run `docker-devenv exec` to get a shell prompt inside the devenv container. From there you can use the installed toolchain to do your dependencies management without caring what version of that toolchain is installed on the host.

Run `docker-devenv monitor` to what your project directory for changes and do a kill/build/test/run continuously monitoring loop. (For now you will still need to manually restart it to pick up changes to your devenv container however.)

Run `docker-devenv test` if you just want to run the test suite.

Replace all the different invocations of your app in staging, production, development with a single `docker-devenv run`, replace your CI invocation with just `docker-devenv test`. Replace all your build commands with just `docker-devenv build`.

All that's left then is to admire how much simpler your life has become.

And for your next project, even in a different language, you do the configuration once and then you're back to using the same commands and muscle memory: even if you come back to it six months later, you still know how to build it, test it, run it.

## Commands

### build

`docker-devenv build`

Builds the devenv and runtime images.

### run

Creates a temporary container from the devenv image and runs it. Within that container it then runs a temporary container made from the runtime image.

### test

Creates a temporary container from the devenv image and runs the test suite within it.

### monitor

Create a temporary container from the devenv image and run the build, test suite, and finally runtime image within it. Monitor the project filesystem for changes and kill and relaunch the build/test/run sequence after three seconds of no further filesystem changes.

### exec

Runs the given command, or a shell by default, on the devenv container.

### rexec

Runs the given command, or a shell by default, on the runtime container.

### version

Show version information and exit.

### help

Show supported subcommands and simple help for them.

### update

Do an in-place update of the docker-devenv script with the most recent version.

### plugins

Lists any plugins you've installed for docker-devenv.

## Dockerfile requirements

Easiest way to get your `Dockerfile` up and running is to look in the examples dir and copy from there, but here's the minimum requirements if you insist on doing it the hard way.

For brevity these examples assume a multi-stage build that pulls out the base stage common to all stages into a stage called `common` and one used for build/dev that includes called `toolchain`, you're not required to do this, but it makes the example shorter.

The example also uses alpine and assumes a nodejs project using npm, but neither of these are requirements, they're just to give you something real-world to see how the pieces fit together.

The only requirement here is that it should accept an arg called `APP`.

```Dockerfile
# syntax=docker/dockerfile:1
#
# "common" intermediate image used by both build and run stage.
#
FROM node:lts-alpine AS common
WORKDIR /app

ARG APP

#
# "toolchain" intermediate image used for build and development-environment images.
#
FROM common AS toolchain

RUN --mount=type=cache,target=/var/cache/apk apk add yq

COPY package*.json .
RUN --mount=type=cache,target=node_modules,sharing=private \
    npm install --verbose --include=dev --install-links --fund=false --update-notifier=false
COPY . .
```

### Add a build stage

You need a build stage to run the build of the runtime image. The build stage should set the `BUILDING_APP` env var to the name of your app, as configured in the `env` file, so that `docker-devenv` can detect it.

It should just run `docker-devenv build`, and you if you need to do any thing here you should ideally put all your build steps in the `custom-build` file rather than in the `Dockerfile` although nothing should break if you don't.

```Dockerfile
#
# "build" intermediate image used for, er, the build
#
FROM toolchain AS build

ARG BUILDING_APP=$APP

RUN --mount=type=cache,target=node_modules,sharing=private ./bin/docker-devenv build
```

### Add a devenv stage

You need a devenv stage to allow a developer to manually run toolchain commands to manipulate the source code. The devenv stage should set the `DEVENV_APP` env var to the name of your app, as configured in the `env` file, so that `docker-devenv` can detect it.

If you want `docker-devenv monitor` behaviour on development machines the default command of the devenv image should be `docker-devenv monitor`, otherwise `docker-devenv run`.

```Dockerfile
#
# "devenv" image, used to do your development in
#
FROM toolchain AS devenv

ENV DEVENV_APP=$APP

# Install additional toolchain for development
RUN --mount=type=cache,target=/var/cache/apk apk add docker inotify-tools

VOLUME /app
CMD ["./bin/docker-devenv", "monitor"]
```

### Finally, you need a runtime stage

To actually run your app, you need a runtime stage to define the runtime image. The runtime stage should set the `RUNNING_APP` env var to the name of your app, as configured in the `env` file, so that `docker-devenv` can detect it.

The default entrypoint should for simplicity run `docker-devenv run`, with the actual app command in `custom-run`, but if for some reason you're unable to run shell commands in the container (no /bin/sh installed for example) you may need to invoke your binary directly.

```Dockerfile
#
# "runtime" image, contains only that stuff necessary to run in deployment
#
FROM common AS runtime

COPY package*.json .
RUN npm install --omit=dev --install-links --fund=false --update-notifier=false

COPY --from=build /app .

ENV RUNNING_APP=$APP
EXPOSE 3000
CMD ["./bin/docker-devenv", "run"]
```

## License

MIT Licensed, see `LICENSE` file for details.
