---
title: Use GitHub Actions to build and push docker image of a Flask App
date: '2022-06-12'
tags: ['Docker','Flask','DevOps']
draft: false
summary: Use GitHub Actions to build and push image to GitHub Package Registry
images: []
layout: PostLayout
canonicalUrl:
---

For development and test, we might need to create rapidly a simple workflow to deploy a web application in a server (like AWS EC2). In this post, I have a Flask application to deploy on a server by using Docker and I will give some of my practices on writing config/workflow files from StackOverflow or medium... which might not be best, but useful for me.

## Setting of gunicorn

Suppose that we have already a Flask application which you can run it by `flask run`. To run it on a server, whether for development or production, you would need to run it by `gunicorn` or `uwsgi` with some specfic configuration. Create a `gunicorn.py` in the root folder of your project, with these settings:

```python
# gunicorn.py
import multiprocessing
import os

if not os.path.exists('logs'):
        os.makedirs('logs')

preload_app = False
bind = '0.0.0.0:5001'

workers = int(multiprocessing.cpu_count() * 2) + 1
worker_class = 'gevent'
threads = int(multiprocessing.cpu_count() * 2) + 1
timeout=300
accesslog='logs/access.log'
errorlog='logs/error.log'
loglevel = "info"

reload = False
```

With this setting, the application will run with `(2 * CPU) + 1` workers and threads. The access logs and error logs will be redirected to the file `logs/*.log` with level of `INFO` for our development/test use case.
To know more about optimizing Gunicorn performance by setting numbers of workers/threads, you can read this post: [Better performance by optimizing Gunicorn config](https://medium.com/building-the-system/gunicorn-3-means-of-concurrency-efbb547674b7).

I suppose that you have already installed `gunicorn` in your virtual environment, so launch your application with

`gunicorn -c python:config.gunicorn <app_name>:app`

with replacing `app_name` by yours.

Check the connection by accessing your application on port 5001 of localhost and check the logs in the `logs` folder. The behavior of application should be same by running it with `flask run`.

## Dockerize your application

We wish to dockerize our application to run it in a container. I suppose you have some basic knowledge of docker.

### Create a `Dockerfile`

Create a `Dockerfile` in the root folder of your project:

```docker
FROM python:3.8

# First, we need to install Pipenv
RUN pip install pipenv

# Then, we need to convert the Pipfile to requirements.txt
COPY Pipfile* /tmp/

RUN cd /tmp && pipenv lock --keep-outdated --requirements > requirements.txt

# Last, we install the dependency and then we can start the Gunicorn.
RUN pip install -r /tmp/requirements.txt

COPY . /tmp/app

WORKDIR /tmp/app

ARG FLASK_CONFIG

ENV FLASK_CONFIG=${FLASK_CONFIG}

CMD ["gunicorn", "-c", "python:config.gunicorn", "flasky:app"]
```

Something might be different from your use case. I use `pipenv` in my project who manages the dependencies by `Pipfile`. I have to convert the `Pipfile` to `requirements.txt`. If you use `virtualenv`, you can replace those lines of code with:

```Docker
# We copy just the requirements.txt first to leverage Docker cache
COPY ./requirements.txt /tmp/requirements.txt

RUN pip install -r /tmp/requirements.txt
```

I have some secret environment variables in my application, so I will try to [pass the variables during build time](https://blog.bitsrc.io/how-to-pass-environment-info-during-docker-builds-1f7c5566dd0e) with `ARG` then use them as **environment variables** with `ENV`.

Try to build the image with

```sh
docker build \
        --build-arg <key1>=<value1> \
        --build-arg <key2>=<value2> \
        -t <image_tag> .
```

### Run the image in a container

Run the image in background with `docker run --name <container_name> -dp 5001:5001 <image_tag>`.  
Use `docker ps` to check the application is running on port 5001 of localhost.

### Check the logs

The logs are saved in the container. To check the logs produced by the application, we have to access to the files in the container.

Run `docker exec -it <container_name> sh` to explore the filesystem.

The `-i` flag keeps input open to the container, and the `-t` flag creates a pseudo-terminal that the shell can attach to. This will run the `sh` shell in the specified container, giving you a basic shell prompt.

To interact with the container, you can use `ls`, `cd`, `cat` or something else to read the logs.

```bash
$ ls
Dockerfile  Pipfile  Pipfile.lock  README.md  __pycache__  app config doc  flasky.py logs  migrations
$ cd logs
$ ls
access.log  error.log
$ cat access.logs
....
```

Use `exit` to quit the pseudo terminal.

To know more about `docker` and some other useful commands, read this article [How To Use docker exec to Run Commands in a Docker Container](https://www.digitalocean.com/community/tutorials/how-to-use-docker-exec-to-run-commands-in-a-docker-container).

## Push the image

I prefer to use the GitHub Actions to execute some CI/CD operations. Here I will build the image to push it to GitHub Package Registry.

### Your personal access token

To login and push packages on [GitHub Package Registry](https://github.com/features/packages), you will need a Personal Access Token. Create one in your GitHub Profile if you don't have one.

Go to [Settings/Developer Settings](https://github.com/settings/tokens), Click "Generate new token", write a note for the token, select an expiration, choose "repo", "write:packages" and "delete:packages in "Select Scopes". Save this token carefully.

![Personal Access Token](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1655556725510_8645.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

### Set Secret keys in your repo

To build our application, we need to

- set the environment variables in GitHub Action secrets, like `FLASK_CONFIG` in my case or some connection strings to database
- set your personal GitHub Token `GH_TOKEN` to login into the GitHub Packages Registry to push your image

To do that, go to the settings of your repository, find "Security - Secrets - Actions", set your GitHub Token and your secret environment variables in "Repository Secrets"

![Repository Secrets](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1655550937342_5643.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

### Write the GitHub Action Workflow

In the root folder of your project, create `.github/workflow/dev.yaml` as the build workflow for the branch `dev`, or name the yaml file as whatever you wish.  

```yaml
name: Build and Push

# only applied in 'dev' branch when push to GitHub
# Action not triggered for commit of README.md
on:
  push:
    branches:
      - "dev"
    paths-ignore:
      - 'README.md'

# fetch environment variables from repository secrets
env:
  FLASK_CONFIG: ${{ secrets.FLASK_CONFIG }}
  DB: $${{ secrets.DB }}

jobs:
  build-image:
    runs-on: ubuntu-latest
    steps:
      # Checkout the code
      - name: Checkout Repo
        uses: actions/checkout@v2
      # Login to docker
      # This can be used for both Docker Hub and
      # GitHub container registry.
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: $GH_TOKEN
      # Build the docker image and push it.
      - name: Build image
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          build-args: FLASK_CONFIG=$FLASK_CONFIG, DB=$DB
          tags: valeeraZ/flask-app:latest
```

Resume:

- Action will be triggered on commits in 'dev' branch and ignore those of 'README.md'
- Use GitHub token to login to GitHub Package Registry
- build image, then push it to the registry

Next post: pull image, run container, update container with WatchTower 
