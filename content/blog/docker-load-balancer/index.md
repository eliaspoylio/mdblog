---
title: Load balancer with Docker containers
date: "2021-02-13 21:16:54"
description: "Load balancer with Docker containers"
---
Layer 7 load balancer using Nginx as reverse proxy.

Code also available at https://github.com/eliaspoylio/docker-loadbalancer

## What is...

### Load balancer?
Distributing tasks over a set of resources. In this case HTTP-requests over three servers using "round robin" algorithm.
### Reverse proxy?
Client makes a request to a reverse proxy server, which forwards the packet to an internal server. Internal server then responds to the proxy server, which forwards the response to the client.

## Prerequisites
- Docker
- NodeJS
- npm

## NodeJS app
For testing purposes we need to create an app for balancing the load to. We'll make an simple Express app to be hosted on a container.

Create `app/package.json` file:
```json
{
    "name": "docker_web_app",
    "version": "1.0.0",
    "description": "Node.js on Docker",
    "author": "First Last <first.last@example.com>",
    "main": "server.js",
    "scripts": {
      "start": "node server.js"
    },
    "dependencies": {
      "express": "^4.17.1"
    }
}
```
Create `app/app.js` file:
```js
'use strict';

const express = require('express');

// Constants
const APP_ID = process.env.APP_ID;
const PORT = process.env.PORT || 5000;

// App
const app = express();
app.get('*', (_, res) => {
    res.send(`App ${APP_ID} listening to port: ${PORT}`)
});

app.listen(PORT, () => {console.log(`App ${APP_ID} listening on port: ${PORT}`)});
```

## Dockerfile for the app
Create `Dockerfile` file:
```Dockerfile
FROM node:14-alpine
# Create app directory
WORKDIR /usr/src/app
# Install app dependencies
COPY app /usr/src/app
RUN npm install
# Run the test app. No need to expose the port, it is done with docker-compose
CMD [ "node", "app.js" ]
```
Just as a good practice add `.dockerignore` file:
```
node_modules
npm-debug.log
```

## Nginx
Create `nginx/nginx.conf` file:
```apacheconf
upstream backend {
    server 172.17.0.1:5001;
    server 172.17.0.1:5002;
    server 172.17.0.1:5003;
 }

server {
    location / {
        proxy_pass http://backend;
    }
}
```
Create `nginx/Dockerfile` file:
```Dockerfile
FROM nginx
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

## Docker compose
How do we set the system up? We need to create a docker-compose file. Create a file called `docker-compose.yaml`:
```yaml
version: '3'

services:
  loadbalancer:
    build: ./nginx
    ports:
      - "8080:80"
  app1:
    image: app
    environment: 
      - APP_ID=1
    ports:
      - "5001:5000"
  app2:
    image: app
    environment: 
      - APP_ID=2
    ports:
      - "5002:5000"
  app3:
    image: app
    environment: 
      - APP_ID=3
    ports:
      - "5003:5000"
```
## Starting the containers
Build the backend application: `docker build -t app .`

Start the Compose: `docker-compose up`

If you now start your browser and go to `localhost:8080` you should see the message from the backend server: `App 1 listening on port: 5000`. The app id should change on every refresh.

After you're done shut down the containers with `ctrl+c` and/or type `docker-compose down` to stop containers and remove containers, networks and images created with the compose.

## Links
http://nginx.org/en/docs/http/load_balancing.html

https://severalnines.com/database-blog/how-does-database-load-balancer-work

https://nodejs.org/en/docs/guides/nodejs-docker-webapp/

https://docs.docker.com/engine/reference/builder/

https://avinetworks.com/glossary/l4-l7-network-services/


