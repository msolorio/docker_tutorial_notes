## Docker Reference Notes

## Parts 1 - 3

#### Run a docker container from Docker Hub
```sh
docker run -d -p 80:80 docker/getting-started
```

`-d` - runs container in detached mode (in the background)

`-p` - allows to setup port mapping between container point and host machine port

#### Build a docker image from a Dockerfile
```sh
docker build -t getting-started .
```

#### Run a docker container from a local image
```sh
docker run -d -p 3000:3000 getting-started
```

#### See list of running container processes with info
will show the container id
```sh
docker ps
```

#### Stop a container
```sh
docker stop <container-id>
```

#### Remove a container
```sh
docker rm <container-id>
```

#### Stop and remove a container
```sh
docker rm -f <container-id>
```

---

## Part 4: Share the Application
 
1. In Docker Hub, create a repository
  - give same name as your image
2. Tag your local image to point to the repository
  - `docker tag <image-name> <docker-username>/<repository-name>`
3. Push your local image to the repository
  - `docker push <docker-username>/<repository-name>`
  - You can leave out the tagname from the command as we didn't give the image a tag and it will default to a tag of `latest`

Now anyone else can run a container based on the image in Docker Hub by running
```sh
docker run -d -p 3000:3000 <docker-username>/<repository-name>
```

---

## Part 5: Persist the DB

When a container is restarted it is rebuilt from the original image
- This means data written to the file system after starting is lost when you restart

#### Volumes
Volumes allows you to mount a filepath in the container to the host machine.
- Allows data written in that filepath to be persisted even arter restart

#### Steps to add a volume

1. Stop the container
```sh
docker rm -f <container-id>
```

2. Create a named volume
```sh
docker volume create <volume-name>
```

3. Rerun the container with the volume mounted
```sh
docker run -d -p 3000:3000 -v <volume-name>:<file-path-to-data> getting-started

# e.g.
docker run -d -p 3000:3000 -v todos-db:/etc/todos getting-started
```

Now you can stop the container and run it again, mounting the same volume, and see the data persist.

---

## Part 6: Using Bind Mount Volumes

With a Bind Mount volume we can specify where on the host we want the data to be stored.

Use Cases
- Can be used to persist data.
- Often used to provide additional data to containers from a host
- Can be used to
  - provide our application source code to the container
  - see the changes immediately in the container

Start a Docker container that is using a Bind Mount to point to the application
- This way when code changes are made, changes are updated in Docker container
- We see changes immediately
```sh
docker run -dp 3000:3000 \
  -w /app -v "$(pwd):/app" \
  node:12-alpine \
  sh -c "yarn install && yarn run dev"
```

Run docker logs to see if server is running
```sh
docker logs -f <container-id>
```

Make a change in the app and see the change immediately after refreshing browser

Build image with new changes
```sh
docker build -t getting-started .
```

---

## Part 7: Multi Container Apps

Keep different parts of the stack in different containers
- database, server, and client in separate containers
- this way each tier can scale independently
- update versions in isolation

#### Container Network
If two containers are on the same network they can talk to each other

Can add a container to a network
- at container startup
- or connect an existing container

#### Connect containers running on the same network
- each container has its own IP address

Create a network
```sh
docker network create todo-app
```

Start a MySQL container and attach it to the network
- provides a network alias to ID the container on the network
```sh
docker run -d \
  --network todo-app --network-alias mysql \
  -v todo-mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=todos \
  mysql:5.7
```

Confirm DB is up and running
```sh
docker exec -it <mysql-container-id> mysql -u root -p
```

```sh
mysql> SHOW DATABASES;
```

Create netshoot container for troubleshooting network and attach to the network
```sh 
docker run -it --network todo-app nicolaka/netshoot
```

Look up IP address of mysql container by passing in **network alias**
```sh
dig mysql
```

For mySQL 8.0 and higher
```sh
mysql> ALTER USER 'root' IDENTIFIED WITH mysql_native_password BY 'secret';
mysql> flush privileges;
```

Start the dev container
- connect to the network
- specify the environment vars
```sh
docker run -dp 3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:12-alpine \
  sh -c "yarn install && yarn run dev"
```

Check logs to see if app is connected to mySQL
```sh
docker logs <container-id>
```

Connect to MySQL db to see if items are written
```sh
docker exec -it <mysql-container-id> mysql -p todos
```

```sh
mysql> select * from todo_items;
```

---

## Part 8: Using Docker Compose

Create a docker compose file for the app container and the mysql container
- notice an entry for each of the configurations passed in when running the `docker run` to start up the containers
```yaml
version: "3.8"

services:
  app:
    image: node:12-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:5.7
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:

```

Start the application stack from the docker compose file
```sh
docker-compose up -d
```

View the logs for the docker-compose startup
```sh
docker-compose logs
```

TIP: Waiting for DB to start up
- For node apps, can use `wait-port` package to wait for the DB

Tear down the app with Docker compose
```sh
docker-compose down
```

Tear down the app and remove the volumes
```sh
docker-compose down --volumes
```

---

## Part 9: Image Building Best Practices

Use `docker scan` to scan image for vulnerabilities

View the layers of an image build from commands from the Dockerfile
```sh
docker image history <image-name>
```

Once a layer is changed, all down stream layers will change too
- the order of commands in the Dockerfile matters

Reorder Dockerfile to copy in package.json and install dependancies before bringing in app files
- this way making a change to app files will not reinstall dependancies

```
# syntax=docker/dockerfile:1
FROM node:12-alpine
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --production
COPY . .
CMD ["node", "src/index.js"]
```

Create `.dockerignore` file in root of app and include `node_modules

Build image and test
```sh
docker build -t getting-started
```

Make a change to an app file
- notice we are pulling in layers from a cache and do not need to re-install node modules

#### React Example

- Build static front end assets with a Node container
- Host on nginx container
- No need to host Node on our destination container

```
# syntax=docker/dockerfile:1
FROM node:12 AS build
WORKDIR /app
COPY package* yarn.lock ./
RUN yarn install
COPY public ./public
COPY src ./src
RUN yarn run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```
