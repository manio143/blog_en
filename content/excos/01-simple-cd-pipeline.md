---
title: 1. Simple CI/CD pipeline
images: []
---

# 1. Simple CI/CD pipeline

Before I start writing code for a new project I want to set up a very simple pipeline for getting it deployed.
I decided to use Docker containers as means of packaging the app since they offer a lot of simplicity once built.
If I was using a cloud platform for running containers it would likely come with some deployment system and a GitHub integration.
But I want to keep this project fairly low cost for now. Instead of using a big cloud provider I'm renting a small VPS for 8 euro a month.

A small server with a static IP address running docker.
A basic setup for starting out. I tried googling container orchestration and everything comes with a lot of upfront configuration.
Except `docker-compose` which has small enough footprint for my needs and I've worked with it before enough to be able to set something up within an hour.
Docker commands can be executed over an SSH connection.

Another thing is dealing with docker images.
I could use a container registry for storing the images, but if I want to iterate quickly I may end up creating a lot of images.
Depending on the registry this comes with adequate costs, so I'm thinking of using it only for releases (however that gets defined later) while managing the short term images directly on the target host.

## Preparing the image

I'm going to skip over the building process for now. The important part is that we end up with a tagged image on our build machine.
Ideally we want to assign a unique build number to the image each time we build the software.
Same build number should be available to the application within the image to be later used for telemetry.

We want to follow the [Semantic Versioning](//semver.org) and one of the options is doing something like `MAJOR.MINOR.PATCH-BUILD+SHA` for continuously building the image (e.g. in a PR or feature branch) and dropping the build info once the change is accepted (e.g. merged into main).
We can leverage something like [GitVersion](https://gitversion.net/) to automate the version calculation.

## Uploading the image

In order to upload the image to the target host we will leverage the `docker save` and `docker load` commands.
They work with a tarball and need an extra program to compress the image in transit (e.g. `gzip`).
We will also leverage docker's integration with SSH.
The command will be as follows ([ref](https://stackoverflow.com/a/62176367)):

```bash
docker save <image:tag> | gzip | DOCKER_HOST=ssh://<user>@<host> docker load
```

In order for this to work the user has to have access to the docker socket.
However, having access to the docker socket means being able to ask docker engine to execute anything (as root).

## Running containers with compose

We can leverage the `DOCKER_HOST` (or docker contexts) to remotely invoke the `docker compose` command and specify the configuration file with the `-f` parameter.

```bash
scp docker-compose.yml <user>@<host>:<path>
DOCKER_HOST=ssh://<user>@<host> docker compose up -d -f <path>
```

## Restricting access

Docker engine is running under the root account which means it's able to do anything on the system.
In particular a user with docker access could run a container and mount the filesystem root of the host in the container.
There's really a lot of space for exploitation and the general advice is to only give access to trusted users.
However, since I'm going to give the CI system (e.g. GitHub Actions) the SSH key to access my server, I want to ensure there's not too much an attacker could do should they obtain the secret key.

I've been looking online and there's generally not much on this topic.
Majority of articles gives solid advice on how to isolate containers from one another or from the host.
Example lists: [OWASP](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Docker_Security_Cheat_Sheet.md), [SecurityArchitect](https://medium.com/@SecurityArchitect/docker-security-settings-for-running-untrusted-trusted-containers-at-the-same-time-88c4ca012726).

From that second list I stumbled over this article on the use of [AppArmor](https://security.theodo.com/en/blog/security-docker-apparmor) which does mention you can apply a restrictive policy to the docker engine process as well ([template](https://github.com/moby/moby/blob/master/contrib/apparmor/template.go)).
I might need to learn a bit more about AppArmor as it's a solid security mechanism.

### Containers

Another set of things we can do is modify the Docker daemon [configuration](https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file) to set some good defaults for the containers.

> Note that any docker configuration can be overridden when running a command and this does not constitute a security measure!

* `"userns-remap": "default"` - makes root user inside a container an unprivileged user when interacting with the host
* `"no-new-privileges": true` - prevent the container from gaining new privileges via setuid or setgid binaries
* `"icc": false` - disable inter-container connectivity over docker0 bridge network (requires creating custom networks)

Additionally we can install [sysbox](https://github.com/nestybox/sysbox) and set the `runtimes` and `default-runtime` properties in the daemon config to use it.

### Leveraging sudoers

Another option I was looking at was to skip giving a user direct access to the Docker socket and instead use a sudoers configuration to provide a set of elevated commands the user can execute.
This means that in the earlier examples instead of using SSH through the Docker client we will use SSH directly.

```bash
docker save <image:tag> | gzip | ssh <user>@<host> sudo docker load
# and then
scp docker-compose.yml <user>@<host>:<path>
ssh <user>@<host> sudo docker compose up -d -f <path>
```

For this we will create a new file under `/etc/sudoers.d/<user>` for our specific user:

```
<user>  ALL=(root:docker) NOPASSWD: /usr/bin/docker load
<user>  ALL=(root:docker) NOPASSWD:SETENV: /usr/bin/docker compose up -d -f *
```

This means the user can impersonate only the root user under the docker group, they will not be asked for password and they are allowed to execute the specified commands.
The `SETENV` flag allows us to pass environment variables with `sudo -E` which can be useful for parameterized compose files.
Note that `*` means any string, so the user can provide extra parameters to the compose command apart from just the file.
To restrict it further we should create a shell script which will validate arguments passed to it and apply them correctly.
The script could also check that the `docker-compose.yml` is not making the containers run with higher privileges than we want.

Afterwards verify the config with `sudo visudo -c` and you may have to run `sudo chmod 0440 /etc/sudoers.d/<user>` to restrict access to this file.

#### Example limiting script

```bash
sudo curl -L -o /usr/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
sudo chmod +x /usr/bin/yq
```

```bash
#!/bin/bash
# docker-compose-up.sh

RUNTIMES=$(yq '.services.*.runtime' ./docker-compose.yml)
if [[ "$RUNTIMES" != null* ]]; then
  echo -e "The docker-compose file tries to override the default runtime\n$RUNTIMES" >&2
  exit 1
fi

docker compose -f $1 up -d
```

_Update_: Adding `--wait --wait-timeout 180` to the last command can allow you to wait until all containers are healthy or fail deployment.

## Disk space cleanup

As we are pushing images onto the server we need to be conscious of how much space they take.
You should strive for building small container images with the minimal set of required dependencies, but even then they can add up and clog up the disk.
Therefore you should look to clean up previous images after deployment.

The docker [prune](https://docs.docker.com/engine/manage-resources/pruning/) command can be used to delete unused resources.
If you're versioning the images such that they contain build number, rerunning CI on a previous commit will anyways create a new image so you can safely remove anything older.
But if you want to keep the last N images for easy rollback to a previous version:

```bash
docker rmi $(docker images -q <repository/image> | tail -n +<N+1>)
```

The name passed in doesn't contain the tag, but starts with the repository (if not from Docker Hub).
The images command returns images in order, latest first.
The tail command prints lines starting from X if we pass `-n X`, so to skip N latest images we pass N+1.

Full script (which takes 1 parameter, if not deletes all images):

```bash
#!/bin/bash
OLD_IMAGES=$(docker images -q $1 | tail -n +4)
if [ -n "$OLD_IMAGES" ]; then
  docker rmi $OLD_IMAGES
fi
# remove dangling images
docker image prune -f
```

## Conclusion

There we have it - a simple CI/CD pipeline which pushes a container image directly into the target host and uses docker compose to instantiate, while keeping in mind the security aspect of allowing external access to your server.

Once I have this up and running on GitHub I will add here a link to the workflow definition.

## Extra: Package update

For the CI portion of this, you should look to keep your dependencies up to date.
You can use something like [Renovate](https://github.com/renovatebot/renovate) or at least [Dependabot](https://github.com/dependabot) or set up your own thing.

For example I used [dotnet-outdated](https://github.com/dotnet-outdated/dotnet-outdated) in a simple GitHub [action](https://github.com/excos-platform/excos/blob/main/.github/workflows/dotnet-package-update.yaml) to update my package dependencies an create PR on a weekly basis.

Btw, I recommend watching this talk: [CI/CD antipatterns](https://www.youtube.com/watch?v=OonABHdHD2I)
