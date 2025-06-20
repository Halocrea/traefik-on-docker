# traefik-on-docker
Infos and config files on how to setup traefik on your server and how to use it

# Notice
This is meant for Docker on Linux, and also requires docker-compose to be installed.
If some of the commands below return an error about missing permissions, then you do it wrong (except if it explicitely says you need special permissions, and in that case, RTFM); it should be done in a folder the user you currently use owns.
In our typical install, all this is intended to be done by the user `docker` into their home directory, `/home/docker`.

# Setup 
```sh
git clone https://github.com/Halocrea/traefik-on-docker.git`

cd traefik-on-docker

touch data/acme.json
chmod 600 acme.json
```

Edit the `docker-compose.yml` file:
- **lines 23 and 28**: replace `example.domain.com` with the DN you want to use for Traefik's dashboard
- **line 24**: replace `USER:PASSWORD` with the proper user and password hash you want when trying to access the dashboard, to do that, follow the steps hereafter: 
    - First off, you will need a user who can `sudo`.
    - then `sudo apt install apache2-utils`.
    - then `echo $(htpasswd -nb <USER> <PASSWORD>) | sed -e s/\\$/\\$\\$/g` replacing `<USER>` and `<PASSWORD>` by a username and password.
    - copy the output of this last command.
    - login as the `docker` user again.
    - Paste the copied value in the docker-compose file to replace the `USER:PASSWORD` line 24.


Now edit the file `data/traefik.yml`:
- **line 18**, replace `your@email.com` by a valid email address of yours.
Save and quit.

Now, create the `proxy` docker network and run this:
```sh

docker network create proxy

cd /home/docker/traefik-on-docker

docker-compose up -d
``` 

# Setting a website to be proxified by Traefik
At the root of your project, you must have a `Dockerfile` setting up how the project will be built into a Docker image.
Then, create a `docker-compose.yml` file with this content:
```YAML
version: "3"

services:
  proxifiedcontainer:
    build: .
    container_name: "<name of your container>"
    restart: always
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.proxifiedcontainer.entrypoints=http"
      - "traefik.http.routers.proxifiedcontainer.rule=Host(`sample.subdomain.com`)"
      - "traefik.http.middlewares.proxifiedcontainer-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.proxifiedcontainer.middlewares=proxifiedcontainer-https-redirect"
      - "traefik.http.routers.proxifiedcontainer-secure.entrypoints=https"
      - "traefik.http.routers.proxifiedcontainer-secure.rule=Host(`sample.subdomain.com`)"
      - "traefik.http.routers.proxifiedcontainer-secure.tls=true"
      - "traefik.http.routers.proxifiedcontainer-secure.tls.certresolver=http"
      - "traefik.http.routers.backoffice-secure.service=proxifiedcontainer"
      - "traefik.http.services.proxifiedcontainer.loadbalancer.server.port=<port>"
      - "traefik.docker.network=proxy"
    networks:
      - internal
      - proxy

networks:
  proxy:
    external: true
  internal:
    external: false
```
**Important values to adjust:**
- **line 5**: you must provide the name of your custom image
- **line 6**: you can name the container the way you want
- **lines 9 *and 20**: replace `<port>` by the port your container exposes stuff _but neither 80 nor 443_ (Traefik is already using those for incoming traffic). For example, for a classical website, it would be `8080:8080`, or for a typical Node app it would be `3000:3000`. If you try one of these and it says that the port is already in use, it may be because another container already uses it, so you have to expose (in your project, in the `Dockerfile` and here) another port.
- lines 13 and 17: replace `sample.subdomain.com` by your own subdomain. 
- you can replace all occurences of `proxifiedcontainer` by whatever you want, however it should be unique (not something another service/project would use anywhere else). 

Once it's done:
```sh
cd /home/docker/<your-project>/ 

docker-compose up -d
```

If for whatever reason, once the container is up, trying to access your subdomain shows a 504 error, you can try restarting the traefik container:
```sh
cd /home/docker/traefik-on-docker

docker-compose down

docker-compose up -d
```

And that should work!