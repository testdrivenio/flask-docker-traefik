# Dockerizing Flask with Postgres, Gunicorn, and Traefik

## Want to learn how to build this?

Check out the [tutorial](https://testdriven.io/blog/flask-docker-traefik/).

## Want to use this project?

### Development

Build the images and spin up the containers:

```sh
$ docker-compose up -d --build
```

Test it out:

1. [http://flask.localhost:8008/](http://flask.localhost:8008/)
1. [http://flask.localhost:8081/](http://flask.localhost:8081/)

### Production

Update the domain in *docker-compose.prod.yml*, and add your email to *traefik.prod.toml*.

Build the images and run the containers:

```sh
$ docker-compose -f docker-compose.prod.yml up -d --build
```
