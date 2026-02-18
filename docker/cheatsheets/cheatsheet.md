# Docker Cheatsheet

> Quick-reference commands and concepts for Docker.

## Core Commands / Concepts

### Images
```bash
docker build -t myapp:1.0 .        # build image from Dockerfile
docker images                       # list local images
docker pull nginx:alpine            # pull from registry
docker push myrepo/myapp:1.0        # push to registry
docker rmi <image_id>               # remove image
docker image prune                  # remove unused images
```

### Containers
```bash
docker run -d -p 8080:80 --name web nginx   # run detached
docker run -it ubuntu bash                  # run interactive shell
docker ps                                   # list running containers
docker ps -a                                # list all containers
docker stop <name|id>
docker start <name|id>
docker rm <name|id>
docker logs -f <name|id>                    # follow logs
docker exec -it <name|id> bash             # exec into container
```

### Volumes & Networks
```bash
docker volume create mydata
docker run -v mydata:/data nginx
docker network create mynet
docker run --network mynet nginx
```

### Cleanup
```bash
docker system prune -a    # remove all unused objects
```

## Dockerfile Basics

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

## Docker Compose

```yaml
services:
  web:
    build: .
    ports:
      - "8080:80"
  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

## Common Patterns

<!-- Add multi-stage build patterns, health check patterns, etc. here -->
