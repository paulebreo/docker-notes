
# Docker Compose

development workflow
---------------------
```bash
eval $(docker-machine env -u)

dc up -d

dc run --rm web /usr/local/bin/python manage.py collectstatic --noinput
dc run --rm web /usr/local/bin/python manage.py migrate
dc run --rm web /usr/local/bin/python manage.py createsuperuser


# if you just need to restart one microservice/container
dc stop web
dc rm -f web
dc up -d web

```

IMAGE VS BUILD (IN VERSION 2-TYPE COMPOSE FILES)
-----------------------------------------------
The following only applies to version 2-type compose files.
In version 1-type compose files, you can't have `image` and `build` both defined
for a service.

Let's say you define your service with `build` and `image` declarations:
```yaml
version: "2"
services:
  web:
    image: pebreo/myapp:latest
    build: ./web  
```
In version 2-type compose files, ff you have both the `build` and `image` defined, then the the built image will be tagged with `pebreo/myapp:latest`, after you run `dc build`.

If you only had `build` defined then the image will just be name
`<currentdir>_<myservicename>`

If only have `image` defined, then it will only use the image
to spin up a container. You do this when you want to deploy to staging/prod.
You would have to make sure that your `Dockerfile` copies your app
code to the image like this:
```
FROM python3.5
RUN mkdir -p /usr/src/app
COPY code/ /usr/src/app
WORKDIR /usr/src/app
```
And to push to staging/prod you will have to build, tag, and push.
##### Build and tag
```
d build -t pebreo/myapp:staging-0.1 -f staging-Dockerfile `pwd`
d push pebreo/myapp:staging-0.1
```
##### Finally, update the image version in your staging docker-compose file e.g. `staging.yml`
```yaml
version: "2"
services:
  web:  
    image: pebreo/myapp:staging-0.1
```


**Version 1-type** `docker-compose.yml` file 
---------------------------------------
A couple of notes about version 1-type compose files:
* In order to connect services, you need to use `links` declaration.
* The `expose` declaration, exposes that containers port to other containers.
* The `ports` declaration, exposes the container ports to the host. For example,
the declaration `"80:80"` means `"host_port:container_port"`.
* You cannot use `image` and `build` at the same time.
```yaml
web:
  restart: always
  build: ./web
  expose:
    - "8000"
  links:   
    - redis:redis
    - postgres:postgres
  volumes:
    - ./web:/usr/src/app
  env_file: .env
  environment:
    DEBUG : 'true'
  command: /usr/local/bin/gunicorn docker_django.wsgi:application -w 2 -b :8000

nginx:
  container_name: nginxcont
  restart: always
  build: ./nginx/
  ports:
    - "80:80"
  volumes:
    - /www/static
  volumes_from:
    - web
  links:
    - web:web

redis:
  restart: always
  image: redis:latest
  ports:
    - "6379:6379"

postgres:
  restart: always
  image: postgres:9.4
  volumes_from:
    - pgdata
  ports:
    - "5432:5432"

pgdata:
  restart: always
  image: postgres:9.4
  volumes:
    - /var/lib/postgresql
  command: "true"

```

**Version 2-type** `docker-compose.yml` file 
---------------------------------------
A couple notes about version 2-type compose files:
* All services are connected to eachother by default: no need to define `networks`.
* If you want to connect services then you can use the `networks` label.
For example, if you wanted the `postgres` service to only connect to `web` 
then you can make a separate network for it called `db_backend`.
* You can connect to other containers by using the service name of the container you
want to connect to. For example, from the `web` container you can access `nginx`
by doing something like: `nc -w 1 -z nginx 8000`
* The `volumes` declaration means you can define named volumes. Those are
useful because named volumes persist even after you delete containers that connect to it.
```yaml
version: "2"

services:
  web:
    restart: always
    build: 
      context: ./web
      dockerfile: base-django1.8-dev-Dockerfile    
    expose:
      - "8000"
    networks:
      - backend
      - db_backend # this is just for demo purpose
    volumes:
      # during development we mount to local directory to allow livereload when code is changed
      - ./web:/usr/src/app
      # a named volume persists even after we delete the container that attaches to it
      - django-static:/usr/src/app/collectstatic 
    env_file: .env
    environment:
      DEBUG: 'true'
    command: /usr/local/bin/gunicorn docker_django.wsgi:application -w 2 -b :8000

  nginx:
    restart: always
    build: 
      context: ./nginx
      dockerfile: base-nginx-dev-Dockerfile    
    ports:
      - "80:80"
    volumes:
      - /www/static
    volumes_from:
      - web
    networks:
      - backend

  postgres:
    restart: always
    image: postgres:latest
    volumes:
      - pgdata:/var/lib/postgresql/data/
    networks:
      - db_backend # this is just for demo purpose

  redis:
    restart: always
    image: redis:latest
    networks:
      - backend

# named volumes persist even after a container that connects to it is deleted      
volumes:
  django-static:
      driver: local
  pgdata: {}

# use networks to connect/isolate services to/from communicating with eachother
# by default, all services can connect to eachother without networks being defined
networks:
  backend: {}
  db_backend: {}
```

version 3-type compose files
---------------------------
command to deploy in docker cloud:
```
d deploy --compose-file staging.yml
```
you'lle need a `manager node` (swarm manager)
and a `worker node` (swarm worker)

in the example, you need to define under the `deploy` section in the compose file for each service
2 replicas
25% of CPU
25% of RAM
placement in worker node only
restart policy of 3 max attempts with 5 second delay
update policy of one-by-one, 10 second delay and 0.3 failure rate to tolerate during the update
```yaml
version: '3'
services:
redis:
    image: redis:3.2-alpine
    ports:
      - "6379"
    networks:
      - voteapp
    deploy:
      placement:
        constraints: [node.role == manager]
db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - voteapp
    deploy:
      placement:
        constraints: [node.role == manager]
voting-app:
    image: gaiadocker/example-voting-app-vote:good
    ports:
      - 5000:80
    networks:
      - voteapp
    depends_on:
      - redis
    deploy:
      mode: replicated
      replicas: 2
      labels: [APP=VOTING]
      placement:
        constraints: [node.role == worker]
result-app:
    image: gaiadocker/example-voting-app-result:latest
    ports:
      - 5001:80
    networks:
      - voteapp
    depends_on:
      - db
worker:
    image: gaiadocker/example-voting-app-worker:latest
    networks:
      voteapp:
        aliases:
          - workers
    depends_on:
      - db
      - redis
    # service deployment
    deploy:
      mode: replicated
      replicas: 2
      labels: [APP=VOTING]
      # service resource management
      resources:
        # Hard limit - Docker does not allow to allocate more
        limits:
          cpus: '0.25'
          memory: 512M
        # Soft limit - Docker makes best effort to return to it
        reservations:
          cpus: '0.25'
          memory: 256M
      # service restart policy
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      # service update configuration
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: continue
        monitor: 60s
        max_failure_ratio: 0.3
      # placement constraint - in this case on 'worker' nodes only
      placement:
        constraints: [node.role == worker]
networks:
    voteapp:
volumes:
  db-data:
```


https://hackernoon.com/deploy-docker-compose-v3-to-swarm-mode-cluster-4159e9cca712#.vp08vnujk


volumes
--------
creating named volumes allows other containers can share it. also, named volumes have a name so that you can just do a 'docker volume ls |grep volumename' and then do a 'docker volume inspect' which will then tell you where on the host the volume is mounted (usually `/mnt/docker/blah`) 

```yaml

services:
   web

volumes:
   myvolume:
     driver: local # this is the default driver
```

## auto reload

If you want to autoreload you should use the syntax:

```yaml
  volumes:
      - ./web:/usr/src/app 
```

You can't do this in production, so you'll have to take that line out and substitute it with the commented
out line.


