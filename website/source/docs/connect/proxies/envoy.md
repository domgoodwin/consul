---
layout: "docs"
page_title: "Connect - Envoy Integration"
sidebar_current: "docs-connect-proxies-envoy"
description: |-
  Consul Connect has first-class support for configuring Envoy proxy.
---

# Envoy Integration

Consul Connect has first class support for using
[envoy](https://www.envoyproxy.io) as a proxy. Consul configures envoy by
optionally exposing a gRPC service on the local agent that serves [envoy's xDS
configuration
API](https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md).

Currently Consul only supports TCP proxying between services, however HTTP and
gRPC features are planned for the near future along with first class ways to
configure them in Consul.

As an interim solution, [custom envoy configuration](#custom-configuration) can
be specified in [proxy service definition](/docs/connect/proxies.html) allowing
more powerful features of envoy to be used.

## Getting Started

### Installing Envoy

The simplest way to try out envoy with Consul locally is using Docker. Envoy
doesn't release binaries outside of their official Docker image. If you can
build envoy directly then the [`consul connect envoy`
command](/docs/commands/connect/envoy.html) command can be used directly on your
local machine to start envoy, however for this guide we'll use the Docker image.

While the [`consul connect envoy` command](/docs/commands/connect/envoy.html)
supports generating bootstrap config on the host that we could then mount in to
the standard envoy Docker container, it's simpler to be able to use it to run
docker directly which requires both consul and envoy binaries in one container.

Docker 17.05 or higher supports multi-stage builds that make this very simple.
Create a Dockerfile with the following content:

```text
FROM consul:latest
FROM envoyproxy/envoy:v1.8.0
COPY --from=0 /bin/consul /bin/consul
ENTRYPOINT ["dumb-init", "consul", "connect", "envoy"]
```

This takes the consul binary from out latest release image and copies it into a
new image based on the official envoy build.

You can now build this hybrid image locally with:

```text
docker build -t consul-envoy .
```

### Agent Setup

To get a simple example working locally, run a local agent in `-dev` mode.

-> **Note:** `-dev` mode enables the gRPC server on port 8502 by default. For a
production agent you'll need to [explicitly configure the gRPC
port](/docs/agent/options.html#grpc_port).

In order to start a proxy instance, a [proxy service
definition](/docs/connect/proxies.html) must exist on your local agent. The
simplest way to create one is using the [sidecar service
registration](/docs/connect/proxies/sidecar-service.html) syntax.

Create a config file called `envoy_demo.hcl` containing the following service
definitions.

```text
services {
  name = "web"
  port = 8080
  connect {
    sidecar_service {
      port = 18080
      proxy {
        upstreams {
          destination_name = "db"
          local_bind_port = 9191
        }
      }
    }
  }
}
services {
  name = "db"
  port = 9090
  connect {
    sidecar_service {
      port = 19090
    }
  }
}
```

Consul agent can now be started with:

```text
$ consul agent -dev -config-file envoy_demo.hcl
```

Consul clients on the local host will be able to connect to this on the default
addresses and ports, however we will be running envoy in a docker container
which will need to connect back to the agent on the docker host.

Finding the right IP for the host varies depending on platform. If you are using
Docker for Mac or Docker for Windows, the host IP is available via the special
DNS name `host.docker.internal`. For linux hosts this doesn't work currently and
you may need to find the IP of the `docker0` bridge interface as documented
elsewhere. For now we'll use `host.docker.internal` in this example but replace
that with the IP address if needed on linux.

We need to set two environment variables inside the containers for consul and
envoy to be able to connect to the agent on the host. To save repeating them in
the docker run commands later, open a new terminal and export them:

```text
export CONSUL_HTTP_ADDR=host.docker.internal:8500
export CONSUL_GRPC_ADDR=host.docker.internal:8502
```

### Running Envoy

With the above setup, envoy proxies for each service instance can now be run
using:

```text
$ docker run --rm -d -e CONSUL_HTTP_ADDR -e CONSUL_GRPC_ADDR \
  -it --name web-proxy -p18080:18080 \
  consul-envoy -sidecar-for web
3f213a3cf9b7583a194dd0507a31e0188a03fc1b6e165b7f9336b0b1bb2baccb
$ docker run --rm -d -e CONSUL_HTTP_ADDR -e CONSUL_GRPC_ADDR \
  -it --name db-proxy -p19090:19090 \
  consul-envoy -sidecar-for db
d8399b54ee0c1f67d729bc4c8b6e624e86d63d2d9225935971bcb4534233012b
```

We need to expose the public listen ports for the proxies so that the local
agent can health check them and so that they can make mTLS connections to each
other.

To see the output of these you can use `docker logs`. To see more verbose
information you can add `-- -l debug` to the end of the command above. This is
passing the log-level option directly through to envoy. With debug level logs
you should see the config being delivered to the proxy.

### Testing Connectivity

To test basic connectivity, run a dummy TCP service to act as the "db". In this
example we will use a simple TCP echo server. This is started in a container
that shares the network namespace of it's proxy so they can communicate over
loopback. It listens on the port specified in the service definition.

```text
$ docker run -d --name db --network container:db-proxy abrarov/tcp-echo --port 9090
```

Finally, we can simulate acting as the "web" application using a netcat
container that shares the network namespace of the web proxy. The "web" app
declared the `db` service as an upstream with a local bind port of 9191 so we
should be able to connect to the web-proxy on that port and have the connection
proxied through to the `db` echo server:

```text
$ docker run -ti --rm --name web --network container:web-proxy subfuzion/netcat localhost 9191


```