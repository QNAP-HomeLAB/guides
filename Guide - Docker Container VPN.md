Guide - VPN via Docker Container

There are several different VPN containers that would work perfectly fine, including some with included torrent apps, but I have had good luck with the `qmcgaw/gluetun` container. Gluetun is compatible with many different VPN providers, so is a great place to start. The [gluetun wiki](https://github.com/qdm12/gluetun/wiki/Custom-provider) provides detailed instructions on configuring each supported VPN. I personally use Private Internet Access, so that is the example I will show here.

```yaml
version: "3"
services:
  gluetun:
    image: qmcgaw/gluetun
    cap_add:
      - NET_ADMIN
    volumes:
      - gluetun:/gluetun
    environment:
      - VPN_SERVICE_PROVIDER=private internet access
      - OPENVPN_USER=p1234567
      - OPENVPN_PASSWORD=str0nkP@$$
      - SERVER_REGIONS='US Texas'
```

The suggested `docker-compose.yml` does not include a lot of the options I find necessary for ease of use, so here are the additions I suggest including.

Using a custom docker bridge network is a necessity when you want easy intra-container communications. For that reason, I always create this network using the SSH Terminal. Using a non-default docker network allows containers to communicate using the `container_name` instead of requiring the internal docker network IP.

```bash
docker network create --driver "bridge" --opt "encrypted" --scope "local" --subnet "172.27.1.0/24" --attachable "reverse_proxy"
```

Once this network is created using the terminal, verify it exists by running the `docker network ls` command. Now the network needs to be defined in your compose file, and assigned to the docker service (or app). I also add the `container_name` field so we can easily reference this container from other compose files. Change the `openvpn_user` and `openvpn_password` fields for your specific account, and modify the timezone environment variable for your region.


This is the resulting modified compose file:

```yaml
version: "3"

networks:
  reverse_proxy:
    external: true

services:

  app:
    container_name: "vpn"
    image: "qmcgaw/gluetun"
    restart: "unless-stopped"
    cap_add:
      - NET_ADMIN
    networks:
      - "reverse_proxy"
    volumes:
      - gluetun:/gluetun
    environment:
      - TZ=America/Chicago
      - VPN_SERVICE_PROVIDER=private internet access
      - OPENVPN_USER=p1234567
      - OPENVPN_PASSWORD=str0nkP@$$
      - SERVER_REGIONS='US Texas'
```
Save this compose file in its own directory, for example:
`/share/docker/compose/gluetun/docker-compose.yml`

There are various other things you can do to make container management easier, such as bind-mounting all your volumes, and passing "secrets" into the compose file so there is no clear-text sensitive information, but those can be addressed in a separate guide.


Save this `docker-compose.yml` file in a directory that you can access via ssh terminal, or create an app using Container Station and paste in the above compose file. You should be able to run this docker-compose file. I wrote an in-depth guide on folder structure and how to manage docker containers using the terminal in the [QNAP-HomeLAB/docker-scripts](https://github.com/QNAP-HomeLAB/docker-scripts) github repository.

```bash
docker compose -f /share/docker/compose/gluetun/gluetun-compose.yml up -d --remove-orphans
```

Once it is running (check via terminal using `docker ps` to list current running containers), double check that the Container IP is different from your Host IP with these two terminal functions. Both `ipcheck` and `vpncheck` display the Host IP or the Container IP by running a `wget` command to an external website which returns the request IP. You can run these commands individually if you do not want to create the functions each time you log in via SSH.

```bash
ipcheck(){ echo "Container IP: $(docker container exec -it "${*}" wget -qO- ipinfo.io)"; }
vpncheck(){ echo "     Host IP: $(wget -qO- ifconfig.me)" && echo "Container IP: $(docker container exec -it "${*}" wget -qO- ipinfo.io/ip)"; }
```
Individual Host IP command:
```bash
wget -qO- ifconfig.me
```
Individual Container IP command:
```bash
docker container exec -it containername wget -qO- ipinfo.io/ip
```

Now we are ready to set up a container that can only access the internet through this Gluetun VPN connection. A very simple speedtest container from `openspeedtest` works well for this purpose. Note the seven lines I have commented out using a `#` at the front of the line. `depends_on` will only work if this container is a service in the same compose file as the VPN, `networks` and `ports` are only used if the container has it's own network connection, and `network_mode` is necessary so this container attaches to the indicated container for network access.

```yaml
version: "3"

services:
  app:
    container_name: "speedtest"
    image: "openspeedtest/latest"
    network_mode: "container:vpn"
    # networks:
    #   - reverse_proxy
    # ports:
    #   - '3000:3000'
    #   - '3001:3001'
    # depends_on:
    #   - "vpn"
```
Save this compose file in a separate folder from the previously saved gluetun compose file, for example:
`/share/docker/compose/speedtest/docker-compose.yml`

Since the `ports` section cannot be used here, we must define the proper port in the `vpn` container.

```yaml
version: "3"

networks:
  reverse_proxy:
    external: true

services:
  app:
    container_name: "vpn"
    image: "qmcgaw/gluetun"
    restart: "unless-stopped"
    cap_add:
      - NET_ADMIN
    networks:
      - "reverse_proxy"
    ports:
      - '3000:3000'       # speedtest http
      - '3001:3001'       # speedtest https
    volumes:
      - gluetun:/gluetun
    environment:
      - TZ=America/Chicago
      - VPN_SERVICE_PROVIDER=private internet access
      - OPENVPN_USER=p1234567
      - OPENVPN_PASSWORD=str0nkP@$$
      - SERVER_REGIONS='US Texas'

```

Once the new `speedtest` container ports are added to the `vpn` container, you must recreate the vpn container in order for thos ports to be defined. These are the terminal commands to recreate a container:

```bash
docker stop vpn
docker rm vpn
docker compose -f /share/docker/compose/gluetun/gluetun-compose.yml up -d --remove-orphans
docker compose -f /share/docker/compose/speedtest/speedtest-compose.yml up -d --remove-orphans
```

If everything was configured correctly, the `speedtest` container should be available at the local area network address of `nas.lan.ip:3000`, and when you check the container IP, it should show as a VPN IP address instead of your Host IP (public IP). If you created the suggested bash functions, you can type `vpncheck speedtest` and you can quickly see a comparison of your Host and Container IP addresses.

One last thing to add. It is possible to include the `speedtest` application in the same docker compose file as the gluetun vpn container. This will require modification of a single line in the speedtest compose file. Instead of `container:vpn` you will need to use `service:vpn` on the `network_mode` line. The full compose file then looks like this:

```yaml
version: "3"

networks:
  reverse_proxy:
    external: true

services:
  app:
    container_name: "vpn"
    image: "qmcgaw/gluetun"
    restart: "unless-stopped"
    cap_add:
      - NET_ADMIN
    networks:
      - "reverse_proxy"
    ports:
      - '3000:3000'       # speedtest http
      - '3001:3001'       # speedtest https
    volumes:
      - gluetun:/gluetun
    environment:
      - TZ=America/Chicago
      - VPN_SERVICE_PROVIDER=private internet access
      - OPENVPN_USER=p1234567
      - OPENVPN_PASSWORD=str0nkP@$$
      - SERVER_REGIONS='US Texas'

  speedtest:
    container_name: "speedtest"
    image: "openspeedtest/latest"
    network_mode: "service:vpn"
    depends_on:
      - "vpn"
```

Let me know if there are issues with this guide, or you find it difficult to follow. I will post the most updated version in the [QNAP-homelab/docker-guides](https://github.com/QNAP-HomeLAB/guides) repository once this goes live.