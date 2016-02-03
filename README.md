[![GoDoc](https://godoc.org/github.com/docker/engine-api?status.svg)](https://godoc.org/github.com/docker/engine-api)

# Introduction

Engine-api is a set of Go libraries to implement client and server components compatible with the Docker engine.
The code was extracted from the [Docker engine](https://github.com/docker/docker) and contributed back as an external library.

## Components

### Client

The client package implements a fully featured http client to interact with the Docker engine. It's modeled after the requirements of the Docker engine CLI, but it can also serve other purposes.

#### Usage

You can use this client package in your applications by creating a new client object. Then use that object to execute operations against the remote server. Follow the example below to see how to list all the containers running in a Docker engine host:

```go
package main

import (
	"fmt"
	"github.com/docker/engine-api/client"
	"github.com/docker/engine-api/types"
)

func main() {
	defaultHeaders := map[string]string{"User-Agent": "engine-api-cli-1.0"}
	cli, err := client.NewClient("unix:///var/run/docker.sock", "v1.22", nil, defaultHeaders)
	if err != nil {
		panic(err)
	}

	options := types.ContainerListOptions{All: true}
	containers, err := cli.ContainerList(options)
	if err != nil {
		panic(err)
	}

	for _, c := range containers {
		fmt.Println(c.ID)
	}
}
```

#### Middlewares

The engine-api client can use middlewares to handle logic around requests and responses.

A middleware is a function that takes a `transport.Sender` as an argument and returns a new `transport.Sender`. The client creates a chain with those middlewares before sending requests to the remote host.

The [Cookies middleware](client/transport/cookie.go) is an example of a client middleware:

```go
func main() {
	client, _ := client.NewEnvClient()
	cookieJar, _ := cookiejar.New(nil)
	cookies := transport.NewCookieJarMiddleware(cookieJar)

	client.AddMiddlewares(cookies)
}
```

#### Authentication

The engine-api can use different authentication methods, they are implemented in the [client/authn package](client/authn/authn.go). An authentication method implements `authn.AuthResponder`. They are added to the client using `Client.AuthenticateWith`:

```go
func main() {
	client, _ := client.NewEnvClient()
	basicAuthCallback := func(realm string) (user, password string, err error) {
		return "admin", "password", nil
	}
	basicAuth := authn.NewBasicAuth(basicAuthCallback)

	bearerAuthCallback := func(challenge string) (token string, err error) {
		return "token", nil
	}
	bearerAuth := authn.NewBearerAuth(bearerAuthCallback)
	
	client.AuthenticateWith(basicAuth, bearerAuth)
}
```

#### Logger

The engine-api client offers a Logger interface that a caller can supply. By default, the client uses a [silent logger](client/logger/logger.go) that doesn't produce any output. [Logrus](https://godoc.org/github.com/Sirupsen/logrus), for instance, satisfies this interface.

```go
func main() {
	client, _ := client.NewEnvClient()
	client.SetLogger(logrus.StandardLogger())
}
```

### Types

The types package includes all typed structures that client and server serialize to execute operations.

### Server

The server package includes API endpoints that applications compatible with the Docker engine API can reuse. It also provides useful middlewares and helpers to handle http requests.

This package is still pending to be extracted from the Docker engine.

## Developing

engine-api requires some minimal libraries that you can download running `make deps`.

To run tests, use the command `make test`. We use build tags to isolate functions and structures that are only available for testing.

To validate the sources, use the command `make validate`.

## License

engine-api is licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for the full license text.
