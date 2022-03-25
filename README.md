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
