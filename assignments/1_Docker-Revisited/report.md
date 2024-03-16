# Lab Report: Docker revisited

## Assignment description

The goal of the assignment was to have a first experience/refresher with the basics of Docker. By trying to run a small application in a container, we explored multiple key concepts. These key concepts are:

- Building an image
- Running a container
- Docker compose files
- Optimizing images

At last, we also saw how to push an image in the cloud and how ran containers based on those images. This was made possible through DockerHub.

## Proof of work done

### 1. Configuring Portainer

As can be seen from the images below, Portainer was configured succesfully through its dashboard:

![Portainer_configuration_01](./img/01.png)

![Portainer_configuration_02](./img/02.png)

### 2. Create a Docker image for a simple web application

To create the dockerfile, I've first decided to install nano as follow: `sudo dnf install nano`. After this, I've used `nano` to create a Dockerfile with the following content:

```docker
FROM node:18
WORKDIR /app
COPY index.js package.json README.md test.spec.js yarn.lock .
COPY persistence ./persistence
RUN yarn install --frozen-lockfile
EXPOSE 3000
CMD ["yarn", "start"]
```

Thereafter, I've build the image and constructed the container as follow:

```console
docker build -t webapp . # Building docker image
docker run --name webapp -d -p 3000:3000 webapp
```

As can be seen from the console interaction, the container was running succesfully since the application was reachable on both of its endpoints:

```console
[vagrant@dockerlab ~]$ curl 0.0.0.0:3000/animals
[{"id":1,"name":"Goats"},{"id":2,"name":"Lizards"},{"id":3,"name":"Llamas"},{"id":4,"name":"Stick Insects"},{"id":5,"name":"Sugar Gliders"},{"id":6,"name":"Rats"},{"id":7,"name":"Stick Insects"},{"id":8,"name":"Goats"},{"id":9,"name":"Hedgehogs"},{"id":10,"name":"Pigeons"}]
[vagrant@dockerlab ~]$ curl 0.0.0.0:3000/animals/2
{"id":2,"name":"Lizards"}
```

The container was also visible on Portainer:
![Portainer_container_webapp](./img/03.png)

### 3. Create a Docker Compose file

As required, a `docker-compose.yml` file was made with the following content:

```docker
services:

  webapp:
    build: .
    ports:
      - "3000:3000"
```

The service was started with `docker compose up -d` (after having removed the container made in the previous step). Everything worked fine since the application was responding on both of its endpoints:

```console
[vagrant@dockerlab ~]$ curl 0.0.0.0:3000/animals
[{"id":1,"name":"Goats"},{"id":2,"name":"Lizards"},{"id":3,"name":"Llamas"},{"id":4,"name":"Stick Insects"},{"id":5,"name":"Sugar Gliders"},{"id":6,"name":"Rats"},{"id":7,"name":"Stick Insects"},{"id":8,"name":"Goats"},{"id":9,"name":"Hedgehogs"},{"id":10,"name":"Pigeons"}]
[vagrant@dockerlab ~]$ curl 0.0.0.0:3000/animals/2
{"id":2,"name":"Lizards"}
[vagrant@dockerlab ~]$ curl 0.0.0.0:3000/animals/1
{"id":1,"name":"Goats"}
```

### 4. Backup the database

When _entering_ inside the container, there was indeed a `database` folder that was not present om my host system.
![Database_missing_host](./img/04.png)

The `docker-compose.yml` file was then modified as such:

```docker
services:

  webapp:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - ./database:/app/database
```

Afterwards, `docker compose up -d` was again executed and the container was restarted. From this point, the `database` folder was present on my host system and the _Fake data generated_ message was not to be seen anymore.

![Database_present_host](./img/05.png)

```console
yarn run v1.22.19
$ node index.js
Fake data generated
SQLite database initialized
Server running on port 3000
yarn run v1.22.19
$ node index.js
SQLite database initialized
Server running on port 3000
```

### 5. Add a database service

The `docker-compose.yml` file was modified as such:

```docker
services:

  webapp:
    build: .
    environment:
      MONGO_URL: mongodb://database:27017
    ports:
      - "3000:3000"
    volumes:
      - ./database:/app/database

  database:
    image: mongo:4.4.18
    depends_on:
      - webapp
    ports:
      - "27017:27017"
```

After having executed `docker compose up -d`, the used database was indeed a MongoDB database. This could be observed from the message the application printed:

```console
yarn run v1.22.19
$ node index.js
MongoDB database initialized
Server running on port 3000
```

Moreover, when looking at the HTTP header of a reponse, it could also be seen that the application was using a MongoDB database:

```console
[vagrant@dockerlab webapp]$ curl 0.0.0.0:3000/animals -I
HTTP/1.1 200 OK
X-Powered-By: Express
X-Database-Used: MongoDB
Content-Type: application/json; charset=utf-8
Content-Length: 267
ETag: W/"10b-Yx06dQgq5RXM6QPwhqCy8Loj3uI"
Date: Mon, 02 Oct 2023 13:11:02 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

### 6. Backup the database

The `docker-compose.yml` file was modified as such:

```docker
services:

  webapp:
    build: .
    environment:
      MONGO_URL: mongodb://database:27017
    ports:
      - "3000:3000"
    volumes:
      - ./database:/app/database

  database:
    image: mongo:4.4.18
    depends_on:
      - webapp
    volumes:
      - mongo_db:/data/db
    ports:
      - "27017:27017"

volumes:
  mongo_db:
```

The named volume was created as a result of `docker compose up -d`:

```console
[+] Running 3/3
 ✔ Volume "webapp_mongo_db"     Created                                    0.0s
 ✔ Container webapp-webapp-1    Running                                    0.0s
 ✔ Container webapp-database-1  Started                                    1.2s
```

As additional verification, this was verified on Portainer:

![named_volume_portainer](./img/06.png)

### 7. Optimizing the Docker image

First, the `.dockerignore` file was made:

```docker
node_modules

Dockerfile
.dockerignore
docker-compose.yml

database/
```

Then, the Dockerfile was modified as such:

```docker
FROM node:18
COPY package.json yarn.lock .
RUN yarn install --frozen-lockfile
COPY . .
EXPOSE 3000
```

### 8. Testing the application

The docker-compose file was modified as such:

```docker
services:

  webapp:
    build: .
    environment:
      MONGO_URL: mongodb://database:27017
    ports:
      - "3000:3000"
    volumes:
      - ./database:/app/database

  database:
    image: mongo:4.4.18
    depends_on:
      - webapp
    volumes:
      - mongo_db:/data/db
    ports:
      - "27017:27017"

  testing:
    build: .
    environment:
      API_URL: http://webapp:3000
    depends_on:
      - webapp

volumes:
  mongo_db:
```

Once the testing container was started, the tests were executed inside of it. They were al succesful:

```console
root@bc3f27f7c76e:/app# yarn test
yarn run v1.22.19
$ mocha test.spec.js


  Animals
    ✔ should 200 and return all animals (203ms)
    ✔ should 200 and return a single animal (79ms)
    ✔ should 404 and return an error when the animal is not found (57ms)


  3 passing (403ms)

Done in 2.23s.
root@bc3f27f7c76e:/app#
```

### 9. Pushing the Docker image to Docker Hub

As required, the image was succesfully pushed unto DockerHub as can be seen from the screen below:

![named_volume_portainer](./img/07.png)

Also, pulling the image was done succesfully:

```console
[vagrant@dockerlab webapp]$ docker pull dani450/webapp
Using default tag: latest
latest: Pulling from dani450/webapp
167b8a53ca45: Already exists
b47a222d28fa: Already exists
debce5f9f3a9: Already exists
1d7ca7cd2e06: Already exists
94c7791033e8: Already exists
c4aa25333915: Already exists
1a9bab107458: Already exists
f386febadd9d: Already exists
6ddac5656eea: Already exists
7a991ed4fd44: Already exists
bf635c9dec8e: Already exists
72df9ffab446: Already exists
Digest: sha256:ddd2aabf449af1680fc18aa6c7adce8cdd03540d0e30579f3fafc9c5060ccd42
Status: Downloaded newer image for dani450/webapp:latest
docker.io/dani450/webapp:latest
```

The `docker-compose.yml` file was then altered as such:

```docker
services:

  webapp:
    image: dani450/webapp
    environment:
      MONGO_URL: mongodb://database:27017
    ports:
      - "3000:3000"
    volumes:
      - ./database:/app/database

  database:
    image: mongo:4.4.18
    depends_on:
      - webapp
    volumes:
      - mongo_db:/data/db
    ports:
      - "27017:27017"

  testing:
    image: dani450/webapp
    environment:
      API_URL: http://webapp:3000
    depends_on:
      - webapp

volumes:
  mongo_db:
```

After running `docker compose up -d`, the container were indeed using the images from DockerHub. Also, the tests still ran succesfully.

```console
[vagrant@dockerlab webapp]$ docker ps -a
CONTAINER ID   IMAGE                           COMMAND                  CREATED         STATUS         PORTS                                           NAMES
04be79c60559   dani450/webapp                  "docker-entrypoint.s…"   7 minutes ago   Up 7 minutes   3000/tcp                                        webapp-testing-1
b75770cce986   dani450/webapp                  "docker-entrypoint.s…"   7 minutes ago   Up 7 minutes   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp       webapp-webapp-1
99f619310f75   mongo:4.4.18                    "docker-entrypoint.s…"   8 hours ago     Up 8 hours     0.0.0.0:27017->27017/tcp, :::27017->27017/tcp   webapp-database-1
cb9b55a8b17a   portainer/portainer-ce:latest   "/portainer"             12 days ago     Up 2 days      8000/tcp, 9000/tcp, 0.0.0.0:9443->9443/tcp      portainer
```

```console
root@04be79c60559:/app# yarn test
yarn run v1.22.19
$ mocha test.spec.js


  Animals
    ✔ should 200 and return all animals (73ms)
    ✔ should 200 and return a single animal
    ✔ should 404 and return an error when the animal is not found


  3 passing (173ms)

Done in 1.38s.
```

## Cheat sheet expansion

The cheat sheet was expanded with following commands

| Task                        | Command                                                |
| :-------------------------- | :----------------------------------------------------- |
| Go inside a container       | `docker exec -it CONTAINER bash`                       |
| Build docker image          | `docker build -t TAG .`                                |
| Create container from image | `docker run --name CONTAINER_NAME -d -p PORT:PORT TAG` |

## Evaluation criteria

Provide a brief explanation for each box you **didn't** check.

- [x] Show that you created a Docker image for the API
- [x] Show that you can start the API using the SQLite database
- [x] Show that you can start the API using the MongoDB database
- [x] Show that you can access the API on port 3000 on the VM
- [x] Show that you optimized the Docker image size
- [x] Show all running containers in the Portainer dashboard
- [x] Show that all tests are passing
- [x] Show that you pushed the Docker image to Docker Hub and that you can pull it from Docker Hub
- [x] Show that you wrote an elaborate lab report in Markdown and pushed it to the repository
- [x] Show that you updated the cheat sheet with the commands you need to remember

## Issues

- When trying to build my docker image, I always got a timeout (see console interaction below). This was due to a problem on Docker's part (see image). This got solved on its own the following day.

  ```console
  [vagrant@dockerlab webapp]$ docker build -t webapp .
  [+] Building 10.2s (3/3) FINISHED                                                                                        docker:default
   => [internal] load build definition from Dockerfile                                                                               0.0s
   => => transferring dockerfile: 196B                                                                                               0.0s
   => [internal] load .dockerignore                                                                                                  0.0s
   => => transferring context: 2B                                                                                                    0.0s
   => ERROR [internal] load metadata for docker.io/library/node:18                                                                  10.0s
  ------
   > [internal] load metadata for docker.io/library/node:18:
  ------
  Dockerfile:1
  --------------------
     1 | >>> FROM node:18
     2 |     WORKDIR /app
     3 |     COPY ./webapp ./
  --------------------
  ERROR: failed to solve: node:18: failed to do request: Head "https://registry-1.docker.io/v2/library/node/manifests/18": dial tcp: looku
  p registry-1.docker.io on 10.0.2.3:53: read udp 10.0.2.15:45194->10.0.2.3:53: i/o timeout
  ```

  ![dockerhub_problem](./img/problem_01.png)

- When having `image: mongo:latest` in my `docker-compose.yml` file, I always got the following error when running `docker compose up -d`:

  ```console
  WARNING: MongoDB 5.0+ requires a CPU with AVX support, and your current system does not appear to have that!
    see <https://jira.mongodb.org/browse/SERVER-54407>
    see also <https://www.mongodb.com/community/forums/t/mongodb-5-0-cpu-intel-g4650-compatibility/116610/2>
    see also <https://github.com/docker-library/mongo/issues/485#issuecomment-891991814>
  ```

  This error was apparantly due to the fact that the latest version of Docker uses AVX, which my current system hasn't. The issue got solved by using an older MongoDB image.

## Reflection

The only real difficulty was developing an understanding for all those new concepts. I had no past experience with Docker and this lab was my first. I had to familiarize myself with concepts such as volumes, layers, images, etc. After this lab, I feel that I have a better understanding of Docker and I am even more interested in learning this technology that gets more important as time goes by.

## Resources

- <https://kifarunix.com/install-virtualbox-guest-additions-on-almalinux/?expand_article=1#install-virtual-box-guest-additions-on-alma-linux-via-command-line>

- <https://superuser.com/questions/1532590/guest-additionals-kernel-headers-not-found-for-target-kernel>

- <https://docs.docker.com/get-started/02_our_app/>

- <https://www.baeldung.com/ops/docker-default-workdir#:~:text=The%20WORKDIR%20instruction%20in%20a%20Dockerfile%20sets%20the%20current%20working,relative%20to%20the%20specified%20directory>

- <https://forums.docker.com/t/what-does-copy-mean/74121/2>

- <https://docs.docker.com/build/guide/layers/>

- <https://stackoverflow.com/questions/70818543/mongo-db-deployment-not-working-in-kubernetes-because-processor-doesnt-have-avx>
