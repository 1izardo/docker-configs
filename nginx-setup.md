# Setting Up NGINX Reverse Proxy

## Create a Docker Network

Type the following command:

```
docker network create nginx-proxy
```

## Install `nginx-proxy`

Make a new directory in your home directory called `nginx-proxy`. Then create `docker-compose.yml` inside this new directory with the following contents:

```
version: "2"

services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    container_name: nginx-proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx-data/certs:/etc/nginx/certs
      - ./nginx-data/vhost.d:/etc/nginx/vhost.d
      - ./nginx-data/html:/usr/share/nginx/html
      - /var/run/docker.sock:/tmp/docker.sock:ro

  nginx-letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: nginx-letsencrypt
    restart: always
    volumes_from:
      - nginx-proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - DEFAULT_EMAIL=email@example.com  # CHANGE THIS!

networks:
  default:
    external:
      name: nginx-proxy
```

Then run the following command:

```
docker-compose up -d
```

If all went well, you should be able to see some sort of page at the IP of your server.
