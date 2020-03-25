# docker-host-internal-poc

https://dev.to/kevbradwick/how-to-setup-a-reverse-proxy-to-your-host-machine-using-docker-mii

Recently, I found myself needing a reverse proxy for my local development environment. I had two applications running on different ports, and I wanted them to appear to be served from a single top level domain. This can be useful when you want cookies shared across sub-domains.

For example, I had a single Django application running on port 8000, and a React frontend running on Webpack dev server on 8080. I wanted them both to be accessible from api-app.localhost and web-app.localhost respectively.

Naturally, I turned to Docker for a solution and this is what I came up with. 

Here is the Docker compose file that uses the Alpine Nginx image to set up the reverse proxy.

```
version: "3"

services:
  # nginx reverse proxy to enable the api and front end to be served from the
  # same host:port.
  # http://api-app.localhost:5000 -> dockerhost:8000
  # http://web-app.localhost:5000 -> dockerhost:8080
  proxy:
    image: nginx:alpine
    ports:
      - "5000:80"
    volumes:
      - "./docker/web.conf:/etc/nginx/conf.d/web.conf:ro"
      - "./docker/api.conf:/etc/nginx/conf.d/api.conf:ro"
```

It references two Nginx configuration files.

Here's web.conf. You'll notice that there are headers defined that are related to the http version and connection upgrade. This enables Websockets to work that Webpack dev server uses for hot reloading.

```
upstream web-app {
  server host.docker.internal:8080;
}

server {
  listen 80;
  listen [::]:80;

  server_name web-app.localhost;

  location / {
    proxy_pass http://web-app;
    proxy_http_version 1.1;
    proxy_redirect     off;
    proxy_set_header   Upgrade $http_upgrade;
    proxy_set_header   Connection "Upgrade";
    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Host $server_name;
  }
}
```

And here is the api.conf.
```
upstream api-app {
  server host.docker.internal:8000;
}

server {
  listen 80;
  listen [::]:80;

  server_name api-app.localhost;

  location / {
    proxy_pass http://api-app;
    proxy_redirect     off;
    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Host $server_name;
  }
}
```

What makes Nginx proxy to your host machine is the use of host.docker.internal in the upstream.

Before starting the containers, there is one thing left to do. Add the two hosts entries to your host file on your host machine. If that's a Linux machine (including macOS) then edit /etc/hosts, and if you are on Windows, edit C:\Windows\System32\drivers\etc\hosts.
```
127.0.0.1 api-app.localhost web-app.localhost
```
And now start the containers!
```
docker-compose up -d
```
When you start the proxy, and both apps are running on your host machine, you'll be able to reach them on http://web-app.localhost:5000 and http://api-app.localhost:5000.

Enjoy!
