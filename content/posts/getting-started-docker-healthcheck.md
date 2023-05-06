---
title: "Getting Started with Docker HEALTHCHECK Command"
date: 2023-05-06T17:04:40+02:00
tags: [Docker, Dockerfile, healthcheck, Monitoring Containers]
---

In this post, we will discuss what Docker `HEALTHCHECK` is, when it should be used, provide a code example to demonstrate how the `HEALTHCHECK` command works and how to debug it.

[The code is available here](https://github.com/Nico769/blog-code-examples/tree/master/getting-started-docker-healthcheck).

## What is Docker HEALTHCHECK

Docker `HEALTHCHECK` is a **command used to test the status of a container in a systematic way**. The command runs periodically, and if it fails, the container is marked as `unhealthy`.

## When should you use Docker HEALTHCHECK?

Docker HEALTHCHECK **should be used to verify whether a containerized application has started successfully or whether a container is still running after the application has encountered a fatal error**.

For example, when running a database container, it's essential to verify that the database is responding to queries. If the database is not responding, the container should be restarted automatically, or an alert should be sent to the system administrator.

## What happens when no Docker HEALTHCHECK is defined

Assume we wrote a `Dockerfile` to containerize a Flask application `app.py` that has a single endpoint, `/killswitch`. This endpoint will simulate an unexpected application error by stopping the Flask application process and setting its exit code to `42`.

_app.py_

```python
from flask import Flask
import os

app = Flask(__name__)

@app.route("/killswitch")
def killswitch():
    # Simulates an application error by stopping
    # the Flask process with an exit code equal to 42
    app.logger.error("Killswitch has been tripped. Exiting...")
    os._exit(42)
```

_Dockerfile_

```dockerfile
FROM python:3.11-bullseye

RUN pip install --no-cache-dir Flask==2.2.3

COPY app.py /

EXPOSE 5000

# Run the Flask application and make it listen on any interface
CMD ["flask", "run", "--host=0.0.0.0"]
```

Let's **see what happens when we don't define** the Docker `HEALTHCHECK` command in the `Dockerfile`. Build the image `no-check` and run it:

```sh
docker build --tag no-check:latest .

docker run -d -p 5000:5000 --name no-check no-check:latest
```

Use the `docker ps` command to monitor the health status of the `no-check` container:

```sh
watch docker ps -a
```

Use the `curl` command to stop the Flask application process:

```sh
curl --location 'http://127.0.0.1:5000/killswitch'
```

You'll notice that the status of the container has changed to `Exited (42)`.

This shows that if we don't define the `HEALTHCHECK` command in the `Dockerfile` then the **status of a container changes according to the exit status code of the process** ran by the `CMD` command in the `Dockerfile`.

## How to use the HEALTHCHECK command in a Dockerfile

Now, let's see how to use the HEALTHCHECK command in a `Dockerfile`. We add the `/ping` endpoint to our Flask application `app.py` that returns an HTTP status code of `200 Ok` when the application is running fine, otherwise `500 Internal Server Error` is returned.

_app.py_

```python
from flask import Flask
import os

app = Flask(__name__)

has_killswitch_tripped = False

@app.route("/killswitch")
def killswitch():
    global has_killswitch_tripped
    has_killswitch_tripped = True

@app.route('/ping')
def ping():
    if has_killswitch_tripped:
        return ('', 500)
    return ('', 200)
```

We also add the following `HEALTHCHECK` command to the `Dockerfile`. The `curl` command sends a GET request to the `/ping` endpoint every 3 seconds, with a timeout of 2 seconds between each request. If any request fails, the command terminates with an exit code equal to 1, indicating a failure.

_Dockerfile_

```dockerfile
FROM python:3.11-bullseye

RUN pip install --no-cache-dir Flask==2.2.3

COPY app.py /

EXPOSE 5000

# Check whether the app is running
HEALTHCHECK --interval=3s --timeout=2s \
    CMD curl --silent --fail http://localhost:5000/ping || exit 1

# Run the Flask application and make it listen on any interface
CMD ["flask", "run", "--host=0.0.0.0"]
```

## How to debug the HEALTHCHECK command

Let's finally see the `HEALTHCHECK` command in action! Build the image `with-check` and run it:

```sh
docker build --tag with-check:latest .

docker run -d -p 5000:5000 --name with-check with-check:latest
```

Instead of using `docker ps`, this time we use `docker inspect` command **to debug the health checks** of the `with-check` container. The command outputs a JSON that we then pipe into [jq](https://stedolan.github.io/jq/), a nifty tool for manipulating JSON data:

```sh
docker inspect --format='{{json .State.Health}}' with-check | jq .
```

Here's an example output of the command:

```json
{
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2023-04-23T17:58:13.019724054+02:00",
      "End": "2023-04-23T17:58:13.152354495+02:00",
      "ExitCode": 0,
      "Output": "trimmed curl output..."
    }
  ]
}
```

Let's focus on the following fields: `Status`, `FailingStreak` and `Log`:

- the `Status` field indicates the current health status of the container;
- the `FailingStreak` field indicates the number of consecutive times that the container has failed the health checks. Since the container is currently `healthy`, this value is equal to 0;
- the `Log` field is an array of the last 5 health check executions for the container. Each element of the array represents a single health check.

It's time to trip our killswitch and see its effects on the container health:

```sh
curl --location 'http://127.0.0.1:5000/killswitch'
```

After a few seconds, the output of `docker inspect` will be similar to the following:

```json
{
  "Status": "unhealthy",
  "FailingStreak": 3,
  "Log": [
    {
      "Start": "2023-04-23T18:05:56.959766485+02:00",
      "End": "2023-04-23T18:05:57.091282926+02:00",
      "ExitCode": 0,
      "Output": "trimmed curl output..."
    },
    {
      "Start": "2023-04-23T18:06:00.095504732+02:00",
      "End": "2023-04-23T18:06:00.222108049+02:00",
      "ExitCode": 0,
      "Output": "trimmed curl output..."
    },
    {
      "Start": "2023-04-23T18:06:03.230734282+02:00",
      "End": "2023-04-23T18:06:03.359595678+02:00",
      "ExitCode": 1,
      "Output": "trimmed curl output..."
    },
    {
      "Start": "2023-04-23T18:06:06.364223733+02:00",
      "End": "2023-04-23T18:06:06.481556982+02:00",
      "ExitCode": 1,
      "Output": "trimmed curl output..."
    },
    {
      "Start": "2023-04-23T18:06:09.49862591+02:00",
      "End": "2023-04-23T18:06:09.648251026+02:00",
      "ExitCode": 1,
      "Output": "trimmed curl output..."
    }
  ]
}
```

The **container is still running** even though its status is now `unhealthy`, since the last 3 health check executions terminated with an exit code of 1. We can manually shut it down and start a new container or let [Docker Swarm](https://docs.docker.com/engine/reference/commandline/swarm/) do the dirty work for us.

Notice that you can **customize the number of consecutive failures** required for bringing the container into the `unhealthy` state by adding the `--retries=N` option to the `HEALTHCHECK` command.

## Conclusion

Docker HEALTHCHECK is an important command that allows us to **define a command to periodically check the health of a running container**. In this post, we went over the basic usage of the HEALTHCHECK command, discussed when to use it, provided a code example of how to use it in a Dockerfile to check whether a Flask application is healthy and how to debug it.

Please drop a comment to let me know if you found this useful!
