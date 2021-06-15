# Dockerizing Flask with Postgres, Gunicorn, and Traefik

In this tutorial, we'll look at how to set up Flask with Postgres and Docker. For production environments, we'll add on Gunicorn, Traefik, and Let's Encrypt.

## Project Setup

Start by creating a project directory:

```bash
$ mkdir flask-docker-traefik && cd flask-docker-traefik
$ python3.9 -m venv venv
$ source venv/bin/activate
(venv)$
```

> Feel free to swap out virtualenv and Pip for [Poetry](https://python-poetry.org/) or [Pipenv](https://pipenv.pypa.io/). For more, review [Modern Python Environments](https://testdriven.io/blog/python-environments/).

Then, create the following files and folders:

```bash
└── services
    └── web
        ├── manage.py
        ├── project
        │   └── __init__.py
        └── requirements.txt
```

Add [Flask](https://flask.palletsprojects.com/en/2.0.x/) to _requirements.txt_:

```text
Flask==2.0.1
```

Install the package:

```bash
(venv)$ pip install -r requirements.txt
```

Next, let's create a simple Flask application in `__init.py__`:

```python
from flask import Flask, jsonify

app = Flask(__name__)


@app.get("/")
def read_root():
    return jsonify(hello="world")
```

Then, to configure the Flask CLI tool to run and manage the app from the command line, add a _manage.py_ file to the "web" directory:

```python
from flask.cli import FlaskGroup

from project import app

cli = FlaskGroup(app)

if __name__ == "__main__":
    cli()
```

Here, we created a new `FlaskGroup` instance to extend the normal CLI with commands related to the Flask app.

Run the server from the "web" directory:

```bash
(venv)$ export FLASK_APP=project/__init__.py
(venv)$ python manage.py run
```

Navigate to [127.0.0.1:5000](http://127.0.0.1:5000/), you should see:

```json
{
  "hello": "world"
}
```

Kill the server once done. Exit from the virtual environment, and remove it as well.

## Docker

Install [Docker](https://docs.docker.com/install/), if you don't already have it, then add a _Dockerfile_ to the "web" directory:

```dockerfile
# Dockerfile

# pull the official docker image
FROM python:3.9.5-slim

# set work directory
WORKDIR /app

# set env variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# copy project
COPY . .
```

So, we started with a `slim`-based [Docker image](https://hub.docker.com/_/python/) for Python 3.9.5. We then set a [working directory](https://docs.docker.com/engine/reference/builder/#workdir) along with two environment variables:

1. `PYTHONDONTWRITEBYTECODE`: Prevents Python from writing pyc files to disc (equivalent to `python -B` [option](https://docs.python.org/3/using/cmdline.html#id1))
1. `PYTHONUNBUFFERED`: Prevents Python from buffering stdout and stderr (equivalent to `python -u` [option](https://docs.python.org/3/using/cmdline.html#cmdoption-u))

Finally, we copied over the *requirements.txt* file, installed the dependencies, and copied over the Flask app itself.

> Review [Docker for Python Developers](https://mherman.org/presentations/dockercon-2018) for more on structuring Dockerfiles as well as some best practices for configuring Docker for Python-based development.

Next, add a *docker-compose.yml* file to the project root:

```yaml
version: '3.8'

services: 
    web:
        build: ./services/web
        command: python manage.py run -h 0.0.0.0
        volumes: 
            - ./services/web/:/app
        ports: 
            - 5000:5000
        environment: 
            - FLASK_APP=project/__init__.py
            - FLASK_ENV=development    
```

> Review the [Compose file reference](https://docs.docker.com/compose/compose-file/) for info on how this file works.

Build the image:

```bash
$ docker-compose build
```

Once the image is built, run the container:

```bash
$ docker-compose up -d
```

Navigate to [http://127.0.0.1:5000/](http://127.0.0.1:5000/) to again view the hello world sanity check.

> Check for errors in the logs if this doesn't work via `docker-compose logs -f`.

## Postgres

To configure Postgres, we need to add a new service to the _docker-compose.yml_ file, set up [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com/), and install [Psycopg2](http://initd.org/psycopg/).

First, add a new service called `db` to _docker-compose.yml_:

```yaml
version: '3.8'

services: 
    web:
        build: ./services/web
        command: bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; python manage.py run -h 0.0.0.0'
        volumes: 
            - ./services/web/:/app
        ports: 
            - 5000:5000
        environment: 
            - FLASK_APP=project/__init__.py
            - FLASK_ENV=development 
            - DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_dev   
        depends_on: 
            - db
            
    db:
        image: postgres:13-alpine
        volumes: 
            - postgres_data:/var/lib/postgresql/data/
        environment: 
            - POSTGRES_USER=hello_flask
            - POSTGRES_PASSWORD=hello_flask
            - POSTGRES_DB=hello_flask_dev

volumes: 
    postgres_data:
```

To persist the data beyond the life of the container we configured a volume. This config will bind `postgres_data` to the "/var/lib/postgresql/data/" directory in the container.

We also added an environment key to define a name for the default database and set a username and password.

> Review the "Environment Variables" section of the [Postgres Docker Hub page](https://hub.docker.com/_/postgres) for more info.

Take note of the new command in the `web` service:

```
bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; python manage.py run -h 0.0.0.0'
```

`while !</dev/tcp/db/5432; do sleep 1` will continue until Postgres is up. Once up, `python manage.py run -h 0.0.0.0`.

Then, add a new file called _config.py_ to the "project" directory, where we'll define environment-specific [configuration](https://flask.palletsprojects.com/config/) variables:

```python
import os


class Config(object):
    SQLALCHEMY_DATABASE_URI = os.getenv("DATABASE_URL", "sqlite://")
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

Here, the database is configured based on the `DATABASE_URL` environment variable that we just defined. Take note of the default value.

Update `__init__.py` to pull in the config on init:

```python
from flask import Flask, jsonify

app = Flask(__name__)
app.config.from_object("project.config.Config")


@app.get("/")
def read_root():
    return jsonify(hello="world")
```

Add [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com/) and [Psycopg2](http://initd.org/psycopg/) to _requirements.txt_:

```text
Flask==2.0.1
Flask-SQLAlchemy==2.5.1
psycopg2-binary==2.8.6
```

Update `__init__.py` again to create a new `SQLAlchemy` instance and define a database model:

```python
from dataclasses import dataclass

from flask import Flask, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config.from_object("project.config.Config")
db = SQLAlchemy(app)


@dataclass
class User(db.Model):
    id: int = db.Column(db.Integer, primary_key=True)
    email: str = db.Column(db.String(120), unique=True, nullable=False)
    active: bool = db.Column(db.Boolean(), default=True, nullable=False)

    def __init__(self, email: str) -> None:
        self.email = email


@app.get("/")
def read_root():
    users = User.query.all()
    return jsonify(users)
```

Using the [dataclass](https://docs.python.org/3/library/dataclasses.html) decorator on the database model helps us serialize the database objects.

Finally, update _manage.py_:

```python
# app/manage.py

from flask.cli import FlaskGroup

from app.main import app, db

cli = FlaskGroup(app)


@cli.command("create_db")
def create_db():
    db.drop_all()
    db.create_all()
    db.session.commit()


if __name__ == "__main__":
    cli()
```

This registers a new command, `create_db`, to the CLI so that we can run it from the command line, which we'll use shortly to apply the model to the database.

Build the new image and spin up the two containers:

```bash
$ docker-compose up -d --build
```

Create the table:

```bash
$ docker-compose exec web python manage.py create_db
```

<blockquote>
<p>Get the following error?</p>
<div class="codehilite"><pre><span></span>sqlalchemy.exc.OperationalError: (psycopg2.OperationalError)
FATAL:  database "hello_flask_dev" does not exist
</pre></div><p>Run <code>docker-compose down -v</code> to remove the volumes along with the containers. Then, re-build the images, run the containers, and apply the migrations.</p></blockquote>

Ensure the `users` table was created:

```bash
$ docker-compose exec db psql --username=hello_flask --dbname=hello_flask_dev

psql (13.3)
Type "help" for help.

hello_flask_dev=# \l
                                        List of databases
      Name       |    Owner    | Encoding |  Collate   |   Ctype    |      Access privileges
-----------------+-------------+----------+------------+------------+-----------------------------
 hello_flask_dev | hello_flask | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres        | hello_flask | UTF8     | en_US.utf8 | en_US.utf8 |
 template0       | hello_flask | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_flask             +
                 |             |          |            |            | hello_flask=CTc/hello_flask
 template1       | hello_flask | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_flask             +
                 |             |          |            |            | hello_flask=CTc/hello_flask
(4 rows)

hello_flask_dev=# \c hello_flask_dev
You are now connected to database "hello_flask_dev" as user "hello_flask".

hello_flask_dev=# \dt
          List of relations
 Schema | Name | Type  |    Owner
--------+------+-------+-------------
 public | user | table | hello_flask
(1 row)

hello_flask_dev=# \q
```

You can check that the volume was created as well by running:

```bash
$ docker volume inspect flask-docker-traefik_postgres_data
```

You should see something similar to:

```bash
[
    {
        "CreatedAt": "2021-06-05T14:12:52Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "flask-docker-traefik",
            "com.docker.compose.version": "1.29.1",
            "com.docker.compose.volume": "postgres_data"
        },
        "Mountpoint": "/var/lib/docker/volumes/flask-docker-traefik_postgres_data/_data",
        "Name": "flask-docker-traefik_postgres_data",
        "Options": null,
        "Scope": "local"
    }
]
```

Navigate to [http://127.0.0.1:5000](http://127.0.0.1:5000). The sanity check shows an empty list. That's because we haven't populated the `users` table. Let's add a CLI seed command for adding sample `users` to the users table in _manage.py_:

```python
from flask.cli import FlaskGroup

from app.main import User, app, db

cli = FlaskGroup(app)


@cli.command("create_db")
def create_db():
    db.drop_all()
    db.create_all()
    db.session.commit()


@cli.command("seed_db") # new
def seed_db():
    db.session.add(User(email="michael@mherman.org"))
    db.session.add(User(email="test@example.com"))
    db.session.commit()


if __name__ == "__main__":
    cli()
```

Try it out:

```bash
$ docker-compose exec web python manage.py seed_db
```

Navigate to [http://127.0.0.1:5000](http://127.0.0.1:5000) again. You should now see:

```json
[
  {
    "active": true,
    "email": "michael@mherman.org",
    "id": 1
  },
  {
    "active": true,
    "email": "test@example.com",
    "id": 2
  }
]
```

## Gunicorn

Moving along, for production environments, let's add [Gunicorn](https://gunicorn.org/), a production-grade WSGI server, to the requirements file:

```text
Flask==2.0.1
Flask-SQLAlchemy==2.5.1
gunicorn==20.1.0
psycopg2-binary==2.8.6
```

Since we still want to use Flask's built-in server in development, create a new compose file in the project root called _docker-compose.prod.yml_ for production:

```yaml
version: '3.8'

services: 
    web:
        build: ./services/web
        command: gunicorn --bind 0.0.0.0:5000 manage:app
        ports: 
            - 5000:5000
        environment: 
            - FLASK_APP=project/__init__.py
            - FLASK_ENV=production
            - DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_prod
        depends_on: 
            - db
    db:
        image: postgres:13-alpine
        volumes: 
            - postgres_data_prod:/var/lib/postgresql/data/
        environment: 
            - POSTGRES_USER=hello_flask
            - POSTGRES_PASSWORD=hello_flask
            - POSTGRES_DB=hello_flask_prod

volumes: 
    postgres_data_prod:
```

Create a new _entrypoint.sh_ in web directory that waits for the postgres instance before starting the gunicorn server.

```sh
#!/bin/sh

echo "Waiting for postgres..."

while ! nc -z db 5432; do
  sleep 0.1
done

echo "PostgreSQL started"

python manage.py create_db
python manage.py seed_db

exec "$@"
```

The script waits for the postgres connection to be ready. Once the connection is up, it runs the `create_db` and `seed_db` commands.

> Note that the `create_db` command drops all tables. This step is just used for demonstration purposes and not recommended for any real life scenarios.

Update the file permissions locally:



```bash
chmod +x services/web/entrypoint.sh
```

Add the entrypoint to the _Dockerfile_:

```Dockerfile
# pull the official docker image
FROM python:3.9.5-slim

# set work directory
WORKDIR /app

# set env variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install system dependencies
RUN apt-get update && apt-get install -y netcat

# install dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# copy project
COPY . .

# run entrypoint.sh
ENTRYPOINT ["/app/entrypoint.sh"]
```

> If you have multiple environments, you may want to look at using a [docker-compose.override.yml](https://docs.docker.com/compose/extends/) configuration file. With this approach, you'd add your base config to a _docker-compose.yml_ file and then use a _docker-compose.override.yml_ file to override those config settings based on the environment.

Take note of the default `command`. We're running Gunicorn rather than the Flask development server. We also removed the volume from the `web` service since we don't need it in production.

Bring [down](https://docs.docker.com/compose/reference/down/) the development containers (and the associated volumes with the -v flag):

```bash
$ docker-compose down -v
```

Then, build the production images and spin up the containers:

```bash
$ docker-compose -f docker-compose.prod.yml up -d --build
```

Verify that the `hello_flask_prod` database was created along with the `users` table. Test out [http://127.0.0.1:5000/](http://127.0.0.1:5000/).

> Again, if the container fails to start, check for errors in the logs via `docker-compose -f docker-compose.prod.yml logs -f`.

## Production Dockerfile

Create a new Dockerfile in the "web " directory called _Dockerfile.prod_ for use with production builds:

```dockerfile
###########
# BUILDER #
###########

# pull official base image
FROM python:3.9.5-slim as builder

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc

# lint
RUN pip install --upgrade pip
RUN pip install flake8==3.9.1
COPY . .
RUN flake8 --ignore=E501,F401 .

# install python dependencies
COPY ./requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt


#########
# FINAL #
#########

# pull official base image
FROM python:3.9.5-slim

# create directory for the app user
RUN mkdir -p /home/app

# create the app user
RUN addgroup --system app && adduser --system --group app

# create the appropriate directories
ENV HOME=/home/app
ENV APP_HOME=/home/app/web
RUN mkdir $APP_HOME
WORKDIR $APP_HOME

# install dependencies
RUN apt-get update && apt-get install -y --no-install-recommends netcat
COPY --from=builder /usr/src/app/wheels /wheels
COPY --from=builder /usr/src/app/requirements.txt .
RUN pip install --upgrade pip
RUN pip install --no-cache /wheels/*

# copy entrypoint.sh
COPY ./entrypoint.sh $APP_HOME

# copy project
COPY . $APP_HOME

# chown all the files to the app user
RUN chown -R app:app $APP_HOME

# change to the app user
USER app

# run entrypoint.prod.sh
ENTRYPOINT ["/home/app/web/entrypoint.sh"]
```

Here, we used a Docker [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) to reduce the final image size. Essentially, `builder` is a temporary image that's used for building the Python wheels. The wheels are then copied over to the final production image and the `builder` image is discarded.

> You could take the [multi-stage build approach](https://stackoverflow.com/a/53101932/1799408) a step further and use a single Dockerfile instead of creating two Dockerfiles. Think of the pros and cons of using this approach over two different files.

Did you notice that we created a non-root user? By default, Docker runs container processes as root inside of a container. This is a bad practice since attackers can gain root access to the Docker host if they manage to break out of the container. If you're root in the container, you'll be root on the host.

Update the `web` service within the docker-compose.prod.yml file to build with _Dockerfile.prod_:

```yaml
web:
    build: 
        context: ./services/web
        dockerfile: Dockerfile.prod
    command: gunicorn --bind 0.0.0.0:5000 manage:app
    ports: 
        - 5000:5000
    environment: 
        - FLASK_APP=project/__init__.py
        - FLASK_ENV=production
        - DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_prod
    depends_on: 
        - db
```

Try it out:

```bash
$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
```

## Traefik

Next, let's add [Traefik](https://traefik.io/traefik/), a [reverse proxy](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/), into the mix.

> New to Traefik? Check out the offical [Getting Started](https://doc.traefik.io/traefik/getting-started/concepts/) guide.


> **Traefik vs Nginx**: Traefik is a modern, HTTP reverse proxy and load balancer. It's often compared to [Nginx](https://www.nginx.com/), a web server and reverse proxy. Since Nginx is primarily a webserver, it can be used to serve up a webpage as well as serve as a reverse proxy and load balancer. In general, Traefik is simpler to get up and running while Nginx is more versatile.
>
> **Traefik**:
>
  1. Reverse proxy and load balancer
  1. Automatically issues and renews SSL certificates, via [Let's Encrypt](https://letsencrypt.org/), out-of-the-box
  1. Use Traefik for simple, Docker-based microservices
>
>
> **Nginx**:
>
  1. Web server, reverse proxy, and load balancer
  1. Slightly [faster](https://doc.traefik.io/traefik/v1.4/benchmarks/) than Traefik
  1. Use Nginx for complex services

Add a new folder `traefik` to the services directory. The project structure should look like:

```bash
├── docker-compose.prod.yml
├── docker-compose.yml
└── services
    ├── traefik
    └── web
```

Add a new file called *traefik.dev.toml* to the "traefik" directory:

```toml
# listen on port 80
[entryPoints]
  [entryPoints.web]
    address = ":80"

# Traefik dashboard over http
[api]
insecure = true

[log]
level = "DEBUG"

[accessLog]

# containers are not discovered automatically
[providers]
  [providers.docker]
    exposedByDefault = false
```

Here, since we don't want to expose the `db` service, we set [exposedByDefault](https://doc.traefik.io/traefik/providers/docker/#exposedbydefault) to `false`. To manually expose a service we can add the `"traefik.enable=true"` label to the Docker Compose file.

Next, update the _docker-compose.yml_ file so that our `web` service is discovered by Traefik and add a new `traefik` service:

```yml
version: '3.8'

services: 
    web:
        build: ./services/web
        command: bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; python manage.py run -h 0.0.0.0'
        volumes: 
            - ./services/web/:/app
        ports: 
            - 5000:5000
        environment: 
            - FLASK_APP=project/__init__.py
            - FLASK_ENV=development 
            - DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_dev   
        depends_on: 
            - db
        labels: 
            - "traefik.enable=true"
            - "traefik.http.routers.flask.rule=Host(`flask.localhost`)"
            
    db:
        image: postgres:13-alpine
        volumes: 
            - postgres_data:/var/lib/postgresql/data/
        environment: 
            - POSTGRES_USER=hello_flask
            - POSTGRES_PASSWORD=hello_flask
            - POSTGRES_DB=hello_flask_dev

    traefik:
        image: traefik:v2.2
        ports: 
            - 80:80
        volumes:
            - "./services/traefik/traefik.dev.toml:/etc/traefik/traefik.toml"
            - "/var/run/docker.sock:/var/run/docker.sock:ro"        

volumes: 
    postgres_data:
```

First, the `web` service is only exposed to other containers on port `5000`. We also added the following labels to the `web` service:

1. `traefik.enable=true` enables Traefik to discover the service
1. ``traefik.http.routers.flask.rule=Host(`flask.localhost`)`` when the request has `Host=flask.localhost`, the request is redirected to this service

Take note of the volumes within the `traefik` service:

1. `./traefik.dev.toml:/etc/traefik/traefik.toml` maps the local config file to the config file in the container so that the settings are kept in sync
1. `/var/run/docker.sock:/var/run/docker.sock:ro` enables traefik to discover other containers

To test, first bring down any existing containers:

```sh
$ docker-compose down -v
$ docker-compose -f docker-compose.prod.yml down -v
```

Build the new development images and spin up the containers:

```sh
$ docker-compose up -d --build
```

Navigate to [http://flask.localhost](http://flask.localhost). You should see:

```json
[
  {
    "active": true,
    "email": "michael@mherman.org",
    "id": 1
  },
  {
    "active": true,
    "email": "test@example.com",
    "id": 2
  }
]
```

You can test via cURL as well:

```bash
$ curl -H Host:flask.localhost http://0.0.0.0
```

Next, checkout the [dashboard](https://doc.traefik.io/traefik/operations/dashboard/) at [flask.localhost:8081](flask.localhost:8081):


![dashboard](dashboard)

Bring the containers and volumes down once done:

```bash
$ docker-compose down -v
```

## Let's Encrypt

We've successfully created a working example of Flask, Docker, and Traefik in development mode. For production, you'll want to configure Traefik to [manage TLS certificates via Let's Encrypt](https://doc.traefik.io/traefik/https/acme/). In short, Traefik will automatically contact the certificate authority to issue and renew certificates.

Since Let's Encrypt won't issue certificates for `localhost`, you'll need to spin up your production containers on a cloud compute instance (like a [DigitalOcean](https://m.do.co/c/d8f211a4b4c2) droplet or an AWS EC2 instance). You'll also need a valid domain name. If you don't have one, you can create a free domain at [Freenom](https://www.freenom.com/).

> We used a [DigitalOcean](https://m.do.co/c/d8f211a4b4c2) droplet along with Docker machine to quickly provision a compute instance with Docker and deployed the production containers to test out the Traefik config. Check out the [DigitalOcean example](https://docs.docker.com/machine/examples/ocean/) from the Docker docs for more on using Docker Machine to provision a droplet.

Assuming you configured a compute instance and set up a free domain, you're now ready to set up Traefik in production mode.

Start by adding a production version of the Traefik config to a file called *traefik.prod.toml*:

```toml
[entryPoints]
  [entryPoints.web]
    address = ":80"
  [entryPoints.web.http]
    [entryPoints.web.http.redirections]
      [entryPoints.web.http.redirections.entryPoint]
        to = "websecure"
        scheme = "https"

  [entryPoints.websecure]
    address = ":443"

[accessLog]

[api]
dashboard = true

[providers]
  [providers.docker]
    exposedByDefault = false

[certificatesResolvers.letsencrypt.acme]
  email = "your@email.com"
  storage = "/certificates/acme.json"
  [certificatesResolvers.letsencrypt.acme.httpChallenge]
    entryPoint = "web"
```

> Make sure to replace `your@email.com` with your actual email address.

What's happening here:

1. `entryPoints.web` sets the entry point for our insecure HTTP application to port 80
1. `entryPoints.websecure` sets the entry point for our secure HTTPS application to port 443
1. `entryPoints.web.http.redirections.entryPoint` redirects all insecure requests to the secure port
1. `exposedByDefault = false` unexposes all services
1. `dashboard = true` enables the monitoring dashboard

Finally, take note of:

```toml
[certificatesResolvers.letsencrypt.acme]
  email = "your@email.com"
  storage = "/certificates/acme.json"
  [certificatesResolvers.letsencrypt.acme.httpChallenge]
    entryPoint = "web"
```

This is where the Let's Encrypt config lives. We defined where the certificates will be [stored](https://doc.traefik.io/traefik/https/acme/#storage) along with the [verification type](https://doc.traefik.io/traefik/https/acme/#the-different-acme-challenges), which is an [HTTP Challenge](https://letsencrypt.org/docs/challenge-types/#http-01-challenge).

Next, assuming you updated your domain name's DNS records, create two new A records that both point at your compute instance's public IP:

1. `flask-traefik.your-domain.com` - for the web service
1. `dashboard-flask-traefik.your-domain.com` - for the Traefik dashboard

> Make sure to replace `your-domain.com` with your actual domain.

Next, update *docker-compose.prod.yml* like so:

```yaml
version: '3.8'

services: 
    web:
        build: 
            context: ./services/web
            dockerfile: Dockerfile.prod
        command: gunicorn --bind 0.0.0.0:5000 manage:app
        expose: 
            - 5000
        environment: 
            - FLASK_APP=project/__init__.py
            - FLASK_ENV=production
            - DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_prod
        depends_on: 
            - db
        labels:
            - "traefik.enable=true"
            - "traefik.http.routers.flask.rule=Host(`flask-traefik.your-domain.com`)"
            - "traefik.http.routers.flask.tls=true"
            - "traefik.http.routers.flask.tls.certresolver=letsencrypt"
    db:
        image: postgres:13-alpine
        volumes: 
            - postgres_data_prod:/var/lib/postgresql/data/
        environment: 
            - POSTGRES_USER=hello_flask
            - POSTGRES_PASSWORD=hello_flask
            - POSTGRES_DB=hello_flask_prod

    traefik:
        build:
            context: ./services/traefik
            dockerfile: Dockerfile.traefik
        ports:
            - 80:80
            - 443:443
        volumes:
            - "/var/run/docker.sock:/var/run/docker.sock:ro"
            - "./traefik-public-certificates:/certificates"
        labels:
            - "traefik.enable=true"
            - "traefik.http.routers.dashboard.rule=Host(`dashboard-flask-traefik.your-domain.com`) && (PathPrefix(`/`)"
            - "traefik.http.routers.dashboard.tls=true"
            - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
            - "traefik.http.routers.dashboard.service=api@internal"
            - "traefik.http.routers.dashboard.middlewares=auth"
            - "traefik.http.middlewares.auth.basicauth.users=testuser:$$apr1$$jIKW.bdS$$eKXe4Lxjgy/rH65wP1iQe1"

volumes:
    postgres_data_prod:
    traefik-public-certificates:
```

> Again, make sure to replace `your-domain.com` with your actual domain.

What's new here?

In the `web` service, we added the following labels:

1. ``traefik.http.routers.flask.rule=Host(`flask-traefik.your-domain.com`)`` changes the host to the actual domain
1. `traefik.http.routers.flask.tls=true` enables HTTPS
1. `traefik.http.routers.flask.tls.certresolver=letsencrypt` sets the certificate issuer as Let's Encrypt

Next, for the `traefik` service, we added the appropriate ports and a volume for the certificates directory. The volume ensures that the certificates persist even if the container is brought down.

As for the labels:

1. ``traefik.http.routers.dashboard.rule=Host(`dashboard-flask-traefik.your-domain.com`)`` defines the dashboard host, so it can can be accessed at `$Host/dashboard/`
1. `traefik.http.routers.dashboard.tls=true` enables HTTPS
1. `traefik.http.routers.dashboard.tls.certresolver=letsencrypt` sets the certificate resolver to Let's Encrypt
1. `traefik.http.routers.dashboard.middlewares=auth` enables `HTTP BasicAuth` middleware
1. `traefik.http.middlewares.auth.basicauth.users` defines the username and hashed password for logging in

You can create a new password hash using the htpasswd utility:

```sh
# username: testuser
# password: password

$ echo $(htpasswd -nb testuser password) | sed -e s/\\$/\\$\\$/g
testuser:$$apr1$$jIKW.bdS$$eKXe4Lxjgy/rH65wP1iQe1
```

Feel free to use an `env_file` to store the username and password as environment variables

```
USERNAME=testuser
HASHED_PASSWORD=$$apr1$$jIKW.bdS$$eKXe4Lxjgy/rH65wP1iQe1
```

Finally, add a new Dockerfile called *Dockerfile.traefik*:

```Dockerfile
# Dockerfile.traefik

FROM traefik:v2.2

COPY ./traefik.prod.toml ./etc/traefik/traefik.toml
```

Next, spin up the new container:

```sh
$ docker-compose -f docker-compose.prod.yml up -d --build
```

Ensure the two URLs work:

1. [https://flask-traefik.your-domain.com](https://flask-traefik.your-domain.com)
1. [https://dashboard-flask-traefik.your-domain.com/dashboard](https://dashboard-flask-traefik.your-domain.com/dashboard)

Also, make sure that when you access the HTTP versions of the above URLs, you're redirected to the HTTPS versions.

Finally, Let's Encrypt certificates have a validity of [90 days](https://letsencrypt.org/2015/11/09/why-90-days.html). Treafik will automatically handle renewing the certificates for you behind the scenes, so that's one less thing you'll have to worry about!

## Conclusion

In this tutorial, we walked through how to containerize a flask application with Postgres for development. We also created a production-ready Docker Compose file, set up Traefik and Let's Encrypt to serve the application via HTTPS, and enabled a secure dashboard to monitor our services.

In terms of actual deployment to a production environment, you'll probably want to use a:

1. Fully-managed database service -- like [RDS](https://aws.amazon.com/rds/) or [Cloud SQL](https://cloud.google.com/sql/) -- rather than managing your own Postgres instance within a container.
1. Non-root user for the services
