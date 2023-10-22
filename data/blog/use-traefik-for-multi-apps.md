---
title: Use Traefik for multi-apps with a same domain
date: '2023-10-22'
tags: ['Docker', 'Flask', 'DevOps']
draft: false
summary: Working with multiple web apps with a same domain for a frontend
images: ['/static/images/traefik-architecture.png']
layout: PostLayout
canonicalUrl:
---

I was working with a front end app which sends request to multiple back ends, and I wanted to use a same domain for all of them. For example, I have a front end app which sends requests to `api1/messages`, `api2/persons`, `api3/shops`... and I want to use a same domain `api.example.com` for all of them with different paths. I found that [Traefik](https://traefik.io/) is a good solution for this case, which can be used as a reverse proxy and load balancer for multiple apps.

## Usecase

Having 2 flask apps:

- `app-1` with a route `/hello`
- `app-2` with a route `world`

I want to use a same domain `api.localhost` for both of them, with different paths as `api.localhost/app-1/hello` and `api.localhost/app-2/world`. The `app-1` and `app-2` are running in different containers in a same network. With Traefik, I don't need to care the ports of each app, I just need to care about the paths.

## Setup

## An example flask app

```python
# app.py for app-1

from flask import Flask

app = Flask(__name__)


@app.route("/hello", methods=["GET"])
def hello():
    return "Hello from app_1"


if __name__ == "__main__":
    app.run(host='0.0.0.0',debug=True, port=5001)
```

This flask app has a route `/hello` which sends back a string `Hello from app_1`. For the port, I set it to 5001, but it can be any port. Here's a simple Dockerfile for this app:

```docker
FROM python:3.11

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 5001

CMD ["python", "app.py"]
```

The `app-2` is similar to `app-1` with a route `/world`, which sends back a string `World from app_2`. I set 5001 port for both apps, but it can be any port whether it's the same or different.

## Traefik

I need to set the `traefik` as a reverse proxy for both apps. I use `docker-compose` to run both apps and traefik in a same network. Here's the `docker-compose.yml`:

```yaml
version: '3'

services:
  reverse-proxy:
    # The official v2 Traefik docker image
    image: traefik:v2.10
    # Enables the web UI and tells Traefik to listen to docker
    command: --api.insecure=true --providers.docker
    ports:
      # The HTTP port
      - '80:80'
      # The Web UI (enabled by --api.insecure=true)
      - '8080:8080'
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock

  app-1:
    build:
      context: ./app_1
      dockerfile: Dockerfile
    container_name: app-1
    labels:
      - 'traefik.enable=true'
      - traefik.http.routers.app-1.rule=Host(`api.localhost`) && PathPrefix(`/app-1`)
      - 'traefik.http.services.app-1.loadbalancer.server.port=5001'
      - 'traefik.http.middlewares.app-1-stripprefix.stripprefix.forceslash=true'
      - 'traefik.http.middlewares.app-1-stripprefix.stripprefix.prefixes=/app-1'
      - 'traefik.http.routers.app-1.middlewares=app-1-stripprefix'

  app-2:
    build:
      context: ./app_2
      dockerfile: Dockerfile
    container_name: app-2
    labels:
      - 'traefik.enable=true'
      - traefik.http.routers.app-2.rule=Host(`api.localhost`) && PathPrefix(`/app-2`)
      - 'traefik.http.services.app-2.loadbalancer.server.port=5001'
      - 'traefik.http.middlewares.app-2-stripprefix.stripprefix.forceslash=true'
      - 'traefik.http.middlewares.app-2-stripprefix.stripprefix.prefixes=/app-2'
      - 'traefik.http.routers.app-2.middlewares=app-2-stripprefix'
```

Let's go through the `docker-compose.yml`:

- the first part `reverse-proxy`: the traefik service. I expose the port 8080 for the traefik dashboard.
- the second part `app-1`: the app-1 service. I set the `traefik` as a reverse proxy for this service with the labels:
  - `traefik.enable=true`: enable the traefik for this service
  - `traefik.http.routers.app-1.rule=Host('api.localhost') && PathPrefix('/app-1')`: set the rule for the router. The router will listen to the request with the host `api.localhost` and the path prefix `/app-1`. To support different paths that point to each app we can use the same rule but add a PathPrefix like this label, which tells Traefik to match a request if it has the same host and the /app-1 prefix.
  - `traefik.http.services.app-1.loadbalancer.server.port=5001`: set the port of the app-1 service. The port can be any port, but it should be the same with the port of the app-1 service which it is listening to.
  - `traefik.http.middlewares.app-1-stripprefix.stripprefix.forceslash=true`: create a middleware with name "app-1-stripprefix". The middleware will strip the prefix from the request before sending it to the app-1 service(because our flask app doesn't have a route like `/app-1/*`). The forceSlash option ensures the resulting stripped path is not the empty string, by replacing it with / when necessary.
    > Removing Prefixes From the Path Before Forwarding the Request, see the [docs](https://doc.traefik.io/traefik/master/middlewares/http/stripprefix/).
  - `traefik.http.middlewares.app-1-stripprefix.stripprefix.prefixes=/app-1`: set the prefix to be stripped from the request for the middleware. The prefix can be any prefix, but it should be the same with the path prefix of the router.
  - `traefik.http.routers.app-1.middlewares=app-1-stripprefix`: set the middleware "app-1-stripprefix" for the router.

The `app-2` service is similar to the `app-1` service, but only with different names for service and its middleware to cut the prefix.

## Run

Launch the docker-compose command, you can see the traefik dashboard at `localhost:8080`:

![traefik dashboard](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=/test.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

We can access the app-1 with the path `api.localhost/app-1/hello` which returns the string `Hello from app_1`. Same with the app-2 with the path `api.localhost/app-2/world` which returns the string `World from app_2`.
