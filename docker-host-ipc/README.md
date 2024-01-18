# Docker Host IPC

This Linux-only container establishes a named pipe that other Docker containers can use to communicate with the Docker host. This requires the use of the `--privileged` flag. Conceptually, this image configures a named pipe that accepts commands, runs them, and (optionally) logs stdout/stderr to a file.

<details title="Why would you do this?"><b>Why would you do this?</b><br/>There are a few different ways for Docker containers to communicate with the host machine. Most are unnecessarily complex, such as using SSH to login to the host. This is a lot of overhead for communication that exists within a single server. It's also possible to setup a web server on the host to serve as a "bridge". That creates even more overhead than SSH communication.<br/><br/>It is far easier, with significantly less overhead/latency, to use a named pipe to communicate. It's not the kind of thing most developers remember how to do off the top of their head though. This image is designed for super-easy implementation while remaning as lightweight as lightest-weight as possible and restricting privileged use to an absolute minimum <i>in a single container</i></details>

A named pipe is established on the Docker _host_, which can be mapped to other Docker containers that utilize the pipe for interprocess communications.

**Run container...**

```sh
docker run --rm \
  --restart always \
  --privileged \
  -e "NAMED_PIPE=/host_path/to/named/pipe" \
  -e "LOG=/host_path/to/pipe.log" \
  author/docker-host-ipc
```

_or as part of docker compose_...

```sh
version: '3'
services:
  docker-ipc:
    image: author/docker-host-ipc
    privileged: true
    restart: always
    environment:
      NAMED_PIPE: "/host_path/to/named/pipe"
      LOG: "/host_path/to/pipe.log"
```

A common pattern we've found to be effective is to create a directory on the host called `/ipc`, then run the docker container as follows:

```shell
mkdir -p /ipc
```

```sh
docker run --rm -d \
  --name ipc \
  --privileged \ # required
  -v "/ipc:/ipc" \ # required
  -e "NAMED_PIPE=/ipc/channel" \  # This line isn't technically necessary since the default is /ipc/channel. It is shown here to illustrate how the container works so you can choose a different named socket (i.e. other than /ipc/channel) if you prefer.
  -e "LOG=/ipc/channel.log" \ # Optional audit log
  author/docker-host-ipc
```

This establishes a named pipe *on the host* at `/ipc/channel` and a log file available at `/ipc/channel.log`.

**Environment Variables**

| Variable          | Required | Default          | Description                                                         |
| :---------------- | :------: | :--------------- | :------------------------------------------------------------------ |
| `NAMED_PIPE`    |    No    | `/ipc/channel` | The location of the named pipe.                                     |
| `LOG`           |    No    | -                | The location of the log file.                                       |
| `MAX_LOG_LINES` |    No    | `25000`        | Truncate the log after this many lines (removes oldest lines first) |

**Volumes**

| Target   |   Required   | Purpose                                           |
| :------- | :-----------: | :------------------------------------------------ |
| `/ipc` | **Yes** | The_host_ directory where the named pipe resides. |

## Using the Named Pipe for IPC Within Docker Containers

Other containers can utilize the pipe once it is created. Consider the following example Docker image:

`Dockerfile` (built as `example`)

```sh
FROM busybox
VOLUME /ipc
RUN echo "ls -l" > /ipc/channel
```

This would be run as follows:

```sh
docker run --rm -v "/ipc:/ipc" -e "LOG=/ipc/channel.log" example
```

The result of `ls -l` (line 4 of Dockerfile) will be available in `/ipc/channel.log` on the host, which will look similar to:

```shell
b
```

Notice the logs are prefixed with the current timetamp.

This is a trivial/contrived example, but it highlights how commands can be executed _on the Docker host_*from within a Docker container*.

## Motivation

This image was initially created to support [auto-letsencrypt-cloudflare](../auto-letsencrypt-cloudflare/README.md). When SSL certificates are renewed, commands are issued to the Docker host to restart containers of the affected services. This prevents the need for other containers to have privileges, making this a more secure option for lightweight interprocess communication within a collection of Docker containers running on the same server.

---

Copyright (c) 2024 Author Software, Inc. All rights reserved, subject to the MIT License.
