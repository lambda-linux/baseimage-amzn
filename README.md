# A minimal Docker Base Image based on Amazon Linux Container Image

baseimage-amzn is a [Docker](https://www.docker.com) [Base Image](https://docs.docker.com/v1.11/engine/reference/glossary/#base-image) that is configured for use within Docker containers. It is based on [Amazon Linux Container Image](https://aws.amazon.com/blogs/aws/new-amazon-linux-container-image-for-cloud-and-on-premises-workloads/), plus:

 * Modifications for Docker-friendliness.
 * Administration tools that are useful in the context of Docker.
 * Mechanisms for running multiple processes.

You can use it as a base for your own Docker images.

baseimage-amzn is available for pulling from [the docker registry!](https://hub.docker.com/r/lambdalinux/baseimage-amzn/)

If you need additional help, please contact us on our [support](https://lambda-linux.io/support/) channels or open a GitHub [Issue](https://github.com/lambda-linux/baseimage-amzn/issues).

baseimage-amzn is inspired by [baseimage-docker](https://github.com/phusion/baseimage-docker) project. We would like to say thank you to Phusion for providing some good ideas and code. We would also like to say thank you to [Amazon Linux Team](https://aws.amazon.com/amazon-linux-ami/) for providing excellent Amazon Linux Container Image.

-----------------------------------------

**Related resources**:
  [Website](https://lambda-linux.io/) |
  [Slack](http://slack.lambda-linux.io/) |
  [Discussion Forum](https://groups.google.com/group/lambda-linux) |
  [Twitter](https://twitter.com/lambda_linux) |
  [Blog](https://lambda-linux.io/blog/) |
  [FAQs](http://lambda-linux.io/faqs/#!/baseimage-amzn-questions)

 **Table of contents**

  * [What's inside the image?](#whats_inside)
    * [Overview](#overview)
    * [A note about SSH server](#about_ssh_server)
    * [baseimage-amzn Version Numbering](#version_numbering)
    * [Inspecting baseimage-amzn](#inspecting)
  * [Using baseimage-amzn as Docker Base Image](#using)
    * [Getting started](#getting_started)
    * [Building and running our Docker Image](#building_and_running)
    * [Adding additional daemons](#adding_additional_daemons)
    * [Running scripts during container startup](#running_startup_scripts)
    * [Environment variables](#environment_variables)
      * [Defining environment variables in our image at build time](#envvar_central_definition)
      * [Using environment variable dump files](#envvar_dumps)
    * [Installing and updating packages inside the container](#updating_packages)
  * [Container administration](#container_administration)
    * [Running a one-shot command in a new container](#oneshot)
    * [Running a command in an existing container](#run_inside_existing_container)
  * [Conclusion](#conclusion)

-----------------------------------------

<a name="whats_inside"></a>
## What's inside the image?

<a name="overview"></a>
### Overview

| Component        | Why is it included? / Remarks |
| ---------------- | ------------------- |
| Amazon Linux | The base system. |
| A init process | baseimage-amzn comes with an init process. Available as `/sbin/my_init`. |
| rsyslog | Syslog daemon is used by various services and applications to log to `/var/log/*` files. |
| logrotate | Rotates and compresses logs files on a regular basis. |
| cron | A cron daemon for cron jobs to run within the container. |
| [runit](http://smarden.org/runit/) | Used for service supervision and management. |
| `setuser` | A tool for running a command as another user. Sets `$HOME` correctly. Available as `/sbin/setuser`. |
| `ll-user` | Image is configured with `ll-user:ll-user` unix user and group, and has a UID/GID pair of 500/500. This UID/GID pair maps correctly to `ec2-user:ec2-user` on Amazon Linux EC2 host. Correct UID/GID mapping between host and container helps avoid permission related issues.|

<a name="about_ssh_server"></a>
### A note about SSH server

We do not ship SSH server in our image. For most users we recommend using [`docker exec`](#run_inside_existing_container) instead.

While it is certainly possible to run SSH server within baseimage-amzn, securing SSH correctly in an operational setting is _non-trivial_. If your use-case _really_ requires running SSH server within baseimage-amzn, please [contact us](https://lambda-linux.io/support/) and we will find a good way to help you.

<a name="version_numbering"></a>
### baseimage-amzn Version Numbering

We would like to give an overview of the version numbering convention that we follow and how that relates to Amazon Linux releases.

Amazon Linux is a _rolling_ distribution. We can think of Amazon Linux as a single river of packages, and images themselves are just snapshots in time. When we run `yum update` our package set gets updated to the tip of this flow.

Releases usually occur in March and September. Release version numbers have the form `20YY.MM`, where `YY` refers to the year, and `MM` refers to the month. For example `2017.03`.

Between major releases, point releases are made by Amazon Linux Team. Point releases have the form `20YY.MM.X`, where `X` is the point release number. For example `2017.03.1`.

baseimage-amzn uses version numbering of the form `20YY.MM-00X`, where `00X` is our point release. For example `2017.03-003`.

We will see the form `20YY.MM-00X` used in the documentation below. You can find the list of baseimage-amzn versions [here](https://hub.docker.com/r/lambdalinux/baseimage-amzn/tags/)

<a name="inspecting"></a>
## Inspecting baseimage-amzn

To look around the image as `root` user, run:

    docker run --rm -t -i lambdalinux/baseimage-amzn:2017.03-003 /sbin/my_init -- /bin/bash -l

    docker run --rm -t -i lambdalinux/baseimage-amzn:<20YY.MM-00X> /sbin/my_init -- /bin/bash -l

To look around the image as `ll-user` user, run:

    docker run --rm -t -i lambdalinux/baseimage-amzn:2017.03-003 /sbin/my_init -- /sbin/setuser ll-user /bin/bash -l

    docker run --rm -t -i lambdalinux/baseimage-amzn:<20YY.MM-00X> /sbin/my_init -- /sbin/setuser ll-user /bin/bash -l

Here `<20YY.MM-00X>` is baseimage-amzn [version number](#version_numbering).

You don't have to download anything manually. The above command will automatically pull baseimage-amzn image from the Docker registry.

<a name="using"></a>
## Using baseimage-amzn as Docker Base Image

<a name="getting_started"></a>
### Getting started

The image is called `lambdalinux/baseimage-amzn` and is available on the Docker registry.

    # Use lambdalinux/baseimage-amzn as base image.
    # See https://hub.docker.com/r/lambdalinux/baseimage-amzn/tags/ for
    # a list of version numbers.
    FROM lambdalinux/baseimage-amzn:<20YY.MM-00X>

    # Use baseimage-amzn's init system
    CMD ["/sbin/my_init"]

    RUN \
      # Update RPM packages
      yum update && \

      # ...put your own build instructions here...

      # Clean up YUM when done
      yum clean all && \
      rm -rf /var/cache/yum/* && \
      rm -rf /tmp/* && \
      rm -rf /var/tmp/*

<a name="building_and_running"></a>
### Building and running our Docker Image

We use [`docker build`](https://docs.docker.com/v1.11/engine/reference/commandline/build/) command to build our Docker Image.

Once the image is built, we can start our container with [`docker run`](https://docs.docker.com/v1.11/engine/reference/commandline/run/) command.

    docker run -d <DOCKER_IMAGE>

Since our `Dockerfile` includes the instruction `CMD ["/sbin/my_init"]`, Docker will start the `my_init` process. `my_init` will set up our [container environment](#environment_variables) and start runit process supervisor. Runit then launches and manage processes inside our container.

We can run `pstree` command within the Docker container to see this behavior in action.

Find the container name of the running Docker container.

    docker ps

Execute `pstree` command in the container.

    docker exec <CONTAINER_NAME> /usr/bin/pstree

    my_init---runsvdir-|-runsv---rsyslogd---2*[{rsyslogd}]
                       `-runsv---crond`

We can stop the running Docker container with [`docker stop`](https://docs.docker.com/v1.11/engine/reference/commandline/stop/) command.

<a name="adding_additional_daemons"></a>
### Adding additional daemons

We can add additional daemons (e.g. our own app) to the image by creating runit entries. We have to write a small shell script which runs our daemon, and runit will keep it running for us, restarting it when it crashes, etc.

Runit requires the shell script to be named `run`. It must be an executable, and should be placed in the directory `/etc/service/<NAME>`.

Here is an example showing how a memcached server runit entry can be made.

Create a file `memcached.sh`. Make sure this file is has execute permission set (`chmod +x`).

    #!/bin/sh
    # `/sbin/setuser memcached` runs the given command as the user `memcached`.
    # If you omit that part, the command will be run as root.
    exec /sbin/setuser memcached /usr/bin/memcached >> /var/log/memcached.log 2>&1

In `Dockerfile`:

    RUN mkdir /etc/service/memcached
    ADD memcached.sh /etc/service/memcached/run

Note that the daemon being executed by the `run` shell script **must not put itself into the background and must run in the foreground**. Daemons usually have a command line flag or a config file option for running in foreground mode.

<a name="running_startup_scripts"></a>
### Running scripts during container startup

The baseimage-amzn init system, `/sbin/my_init`, can run scripts during startup. They are run in the following order if they exist.

  * All executable scripts in `/etc/my_init.d`. The scripts in this directory are executed in alphabetical order.
  * The script `/etc/rc.local`.

All scripts must exit correctly, that is with an exit code 0. If any script exits with a non-zero exit code, the booting will fail.

The following example shows us how to add startup script. This script logs the time of boot to the file `/tmp/boottime.txt`.

Create a file `logtime.sh`. We need to make sure this file has execute permission set (`chmod +x`).

    #!/bin/sh
    date > /tmp/boottime.txt

In `Dockerfile`:

    RUN mkdir -p /etc/my_init.d
    ADD logtime.sh /etc/my_init.d/logtime.sh

<a name="environment_variables"></a>
### Environment variables

When we use `/sbin/my_init` as our main container command, any environment variables defined using `docker run --env` or the [`ENV`](https://docs.docker.com/v1.11/engine/reference/builder/#env) instruction in the `Dockerfile` will be picked up by `/sbin/my_init`. These environment variables will be passed to all child processes, including `/etc/my_init.d` [startup scripts](#running_startup_scripts), runit and [runit managed services](#adding_additional_daemons).

Following example shows this in action.

    $ docker run --rm -t -i \
      --env FOO=bar --env HELLO='my beautiful world' \
      lambdalinux/baseimage-amzn:<20YY.MM-00X> /sbin/my_init -- /bin/bash -l

    [...]

    *** Running /bin/bash -l...
    [root@ff6cbb791855] / # echo $FOO
    bar
    [root@ff6cbb791855] / # echo $HELLO
    my beautiful world
    [root@ff6cbb791855] / #

Here `<20YY.MM-00X>` is baseimage-amzn [version number](#version_numbering).

<a name="envvar_central_definition"></a>
#### Defining environment variables in our image at build time

When `/sbin/my_init` starts up, before running any [startup scripts](#running_startup_scripts), `/sbin/my_init` imports environment variables from the directory `/etc/container_environment`. This directory contains files that are named after the environment variable names. The file contents contain the environment variable values.

`/etc/container_environment` is a good place to define our environment variables at build time. The environment variables defined in `/etc/container_environment` is inherited by all startup scripts and runit services.

For example, here is how we can define an environment variable in our `Dockerfile`.

    RUN echo Apachai Hopachai > /etc/container_environment/MY_NAME

We can verify that it works, as follows:

    $ docker run --rm -t -i \
      <DOCKER_IMAGE> /sbin/my_init -- /bin/bash -l

    [...]

    *** Running /bin/bash -l...
    [root@2a3356297ec4] / # echo $MY_NAME
    Apachai Hopachai
    [root@2a3356297ec4] / #

<a name="envvar_dumps"></a>
#### Using environment variable dump files

Certain services such as Nginx, resets the environment variables of its child processes. When this happens, `/sbin/my_init` provides a way to query the original environment variables that was passed at the time of container launch.

During startup, right after importing environment variables from `/etc/container_environment`, `/sbin/my_init` dumps all its environment variables (that is, all variables imported from `/etc/container_environment` and variables picked up from `docker run --env`) to the following locations.

  * `/etc/container_environment.sh` - Contains the environment variables in bash format. We can source this file from a bash shell script.
  * `/etc/container_environment.json` - Contains the environment variables in JSON format.

Multiple formats makes it easy to query the original environment variables from our favorite programming/scripting language.

Here is an example showing how this works.

    $ docker run --rm -t -i \
      --env FOO=bar --env HELLO='my beautiful world' \
      lambdalinux/baseimage-amzn:<20YY.MM-00X> /sbin/my_init -- /bin/bash -l

    [...]

    *** Running /bin/bash -l...
    [root@42d4b4cd09b3] / # ls /etc/container_environment
    FOO  HELLO  HOME  HOSTNAME  LANG  LC_CTYPE  PATH  PS1  TERM
    [root@42d4b4cd09b3] / # cat /etc/container_environment/FOO; echo
    bar
    [root@42d4b4cd09b3] / # cat /etc/container_environment/HELLO; echo
    my beautiful world
    [root@42d4b4cd09b3] / # cat /etc/container_environment.json; echo
    {"LANG": "en_US.UTF-8", "TERM": "xterm", "PS1": "[\\u@\\h] \\w \\$ ", "HOSTNAME": "42d4b4cd09b3", "LC_CTYPE": "en_US.UTF-8", "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", "HOME": "/", "FOO": "bar", "HELLO": "my beautiful world"}
    [root@42d4b4cd09b3] / # source /etc/container_environment.sh
    [root@42d4b4cd09b3] / # echo $HELLO
    my beautiful world
    [root@42d4b4cd09b3] / # ls -la /etc/container_environment.sh
    -rw-r----- 1 root docker_env 249 Jun 25 11:02 /etc/container_environment.sh
    [root@42d4b4cd09b3] / # ls -la /etc/container_environment.json
    -rw-r----- 1 root docker_env 254 Jun 25 11:02 /etc/container_environment.json

Here `<20YY.MM-00X>` is baseimage-amzn [version number](#version_numbering).

`/etc/container_environment.sh` and `/etc/container_environment.json` files are owned by root and accessible only by the `docker_env` group. To read these files as non-root user, we need to add the non-root user to `docker_env` group.

<a name="updating_packages"></a>
### Installing and updating packages inside the container

Packages can be installed and updated inside the container using `yum install` and `yum update` commands.

We recommend that including the following in `Dockerfile` so that the docker image has the latest packages.

    RUN \
      yum update && \
      yum clean all && \
      rm -rf /var/cache/yum/* && \
      rm -rf /tmp/* && \
      rm -rf /var/tmp/*

<a name="container_administration"></a>
## Container administration

When working with containers, we will encounter situations where we may want to run a command inside a container or login to it. This could be for development, debugging or inspection purposes.

<a name="oneshot"></a>
### Running a one-shot command in a new containers

This section describes how to run a command inside a new container. To run a command inside an existing container see [Running a command inside an existing, running container](#run_inside_existing_container).

baseimage-amzn provides a facility to run a single one-shot command inside a new container the following way.

    docker run <DOCKER_IMAGE> /sbin/my_init -- COMMAND ARGUMENTS ...

This command does the following.

 * Runs all system startup files, such as `/etc/my_init.d/*` and `/etc/rc.local`.
 * Starts all runit services.
 * Runs the specified command.
 * When the specified command exists, stops all runit services.

For example:

    $ docker run lambdalinux/baseimage-amzn:<20YY.MM-00X> /sbin/my_init -- /bin/ls
    *** Running /etc/rc.local...
    *** Booting runit daemon...
    *** Runit started as PID 7
    *** Running /bin/ls...
    bin
    boot

    [...]

    usr
    var
    *** /bin/ls exited with status 0.
    *** Shutting down runit daemon (PID 7)...
    *** Killing all processes...

We can customize how `/sbin/my_init` is invoked. Run `docker run <DOCKER_IMAGE> /sbin/my_init --help` for more information.

The following example runs `/bin/ls` without running the startup files, in quite mode, while running all runit services.

    $ docker run lambdalinux/baseimage-amzn:<20YY.MM-00X> \
      /sbin/my_init --skip-startup-files --quiet -- /bin/ls
    bin
    boot

    [...]

    usr
    var

Here `<20YY.MM-00X>` is baseimage-amzn [version number](#version_numbering).

<a name="run_inside_existing_container"></a>
### Running a command in an existing container

Start Docker container:

    docker run -d <DOCKER_IMAGE>

Find the container name of the running Docker container.

    docker ps

Once we have the container name, we can use [`docker exec`](https://docs.docker.com/v1.11/engine/reference/commandline/exec/) to run commands in the container. For example, to run `echo hello world`:

    docker exec <CONTAINER_NAME> /bin/echo hello world

To open a bash session inside a running container, we need to pass `-t -i` so that a terminal is available.

    docker exec -t -i <CONTAINER_NAME> /bin/bash -l

<a name="conclusion"></a>
## Conclusion

  * Using baseimage-amzn? [Tweet about us](https://twitter.com/share) and [follow us on Twitter](https://twitter.com/lambda_linux).
  * Having problems? Please contact us on any of our [support](https://lambda-linux.io/support) channels or post a GitHub [Issue](https://github.com/lambda-linux/baseimage-amzn/issues).

Thank you for using baseimage-amzn.
