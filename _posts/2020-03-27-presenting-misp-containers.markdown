---
layout: post
title: MISP Containers 🐋
date: '2020-03-27 23:40:00'
tags:
- misp
- cyber threat intelligence
- threat intelligence platforms
- infosharing
- docker
- docker-compose
- containers
- cert
- csirt
- soc
- incident handling
---

Hi guys!
How are you doing?

It has been a long time since my last post.

I would like to present you a project I made in the last few days, during this COVID-19 pandemia that is threatening the entire world.

Here we are! The easiest way to deploy the last version of **MISP**, leveraging the power of containers!

# What do I provide, by now

This setup is made of 12 services built on 6 images as follows.
For the moment images are not in the [Docker Hub](https://hub.docker.com/) or any public registry so you'll have to build them by yourself.

## MISP
This image embeds the MISP source code and configuration files and serves them through *php-fpm*. This container is built in a multi-stage fashion starting from an *Alpine* container.
The first stage fetches the MISP source code from MISP Project's GitHub repository, the second stage prepares the Python virtualenv with all MISP's dependencies, while the third and last puts things together with PHP installing also PHP dependencies.

The versions of the software installed are:

- Alpine Linux container 3.10
- Python 3.7
- PHP 7.4
- Ssdeep 2.13
- Composer 1.10

Some tuning settings for PHP are applied according to MISP's requirements.

I also patched [CakeResque](https://cakeresque.kamisama.me/) to keep it running in foreground, being compliant with Docker's logging approach.
This allows me to give to each worker process its own container, and make workers scale through the container engine.
Since this breaks the native workers management functionality of MISP, within the docker-compose file I decided to share the PID Namespace between workers and MISP services.
Of course, MISP's worker management is definitively broken within Docker Swarm.

Patch is available [here](https://github.com/patriziotufarolo/misp-containers/blob/master/misp/01-cakeresque.patch)

I added two plugins for CakePHP and MISP that allow me to:

- Edit configuration scripts in a consistent way through CLI and scripts using MISP's own functions;
- Verify the readiness of the database service in order to perform some healthchecks wille raising the containers up.

Last but not least, the provided docker-compose specifies also the volumes needed to guarantee persistence.

## MISP modules - *misp-modules*
This image contains all the MISP's enrichment modules and their dependencies.

## Database - *misp-db*
This image is a plain mariadb database image that, on first startup, is initialized with MISP's database schema, grabbed from the misp image throygh a multi-stage approach.

The provided docker-compose specifies also the volumes needed to guarantee persistence.

## Redis - *redis*
Basic Redis image from Docker Hub

## Redis-commander - *rediscommander*
[Redis commander](https://github.com/joeferner/redis-commander) is a Redis web management tool written in Node.JS. I embedded it to look into MISP's redis queues. Feel free to remove it if you don't need it. 

## Frontend - *misp-fe*
This is basically an Nginx image that serves MISP interacting with php-fpm, and redis-commander under the /redis-commander/ path. 
If you don't need redis-commander feel free to remove its definition and rebuild the container.
This is probably the right place (unless you don't have any alternative architecture) to implement HTTPS.

# Details, source code and installation guide
Take a look at the repository here: https://github.com/patriziotufarolo/misp-containers .

Contributions, testing, bug reporting, bug fixing are appreciated!
