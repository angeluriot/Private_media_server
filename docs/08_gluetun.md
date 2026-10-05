# Part 8: [Gluetun](https://github.com/qdm12/gluetun)

### I.&ensp;Create keys

1. In **[Proton VPN](https://account.proton.me/u/1/vpn)**, go to `WireGuard`:

	| Field | Value |
	| --- | --- |
	| Device/certificate name | `mediabox-gluetun` |
	| Platform | `Router` |
	| Level for NetShield blocker filtering | `No filter` |
	| Moderate NAT | ❌ |
	| NAT-PMP (Port Forwarding) | ​✅ |
	| VPN Accelerator | ​✅ |

	Then click `Create` and save the `Private key`

2. Sadly, **[Proton VPN](https://protonvpn.com/)** keys are only valid for 1 year, you'll have to click `Extend` on the same page every year to keep your key valid (you can add a reminder every 11 months in your calendar)

3. On your local computer:

	```bash
	docker run --rm qmcgaw/gluetun:v3.41 genkey
	```

	Save the last line of the output, it's your **[Gluetun](https://github.com/qdm12/gluetun)** API key

<br>

### II.&ensp;Load the TUN module

1. On your local computer, check that the device exists:

	```bash
	ls -l /dev/net/tun
	```

	* [ ] The command should return a line starting with `crw-rw-rw-` and containing `10, 200`

2. Load the module now and at every boot:

	```bash
	sudo modprobe tun
	echo 'tun' | sudo tee /etc/modules-load.d/tun.conf
	cat /etc/modules-load.d/tun.conf
	```

	* [ ] The last command should return `tun`

3. Check that the module is loaded:

	```bash
	lsmod | grep -w tun
	```

	* [ ] The command should return a line starting with `tun`

4. Reboot the local computer and check that steps **1.** and **3.** still work

<br>

### III.&ensp;Update the Docker Compose

1. Add the tokens in the `.env` (you can change the countries depending on your needs):

	```bash
	sudo tee -a /opt/mediaserver/.env > /dev/null <<'EOF'
	PROTON_WIREGUARD_PRIVATE_KEY=<YOUR_PROTON_VPN_PRIVATE_KEY>
	VPN_SERVER_COUNTRIES=Netherlands,Switzerland,Sweden,Romania,Iceland
	GLUETUN_API_KEY=<YOUR_GLUETUN_API_KEY>
	EOF
	```

2. Open `/opt/mediaserver/docker-compose.yml` and add the following content:

	```yaml
	services:
	  jellyfin:
	    ...
	  gluetun:
	    image: qmcgaw/gluetun:v3.41
	    container_name: gluetun
	    restart: unless-stopped
	    cap_add:
	      - NET_ADMIN
	    devices:
	      - /dev/net/tun:/dev/net/tun
	    networks:
	      - media
	    ports:
	      - "127.0.0.1:8000:8000/tcp"
	      - "8080:8080/tcp"
	      - "9696:9696/tcp"
	      - "127.0.0.1:8191:8191/tcp"
	    volumes:
	      - /opt/mediaserver/config/gluetun:/gluetun
	    environment:
	      - PUID=${PUID}
	      - PGID=${PGID}
	      - TZ=${TZ}
	      - VPN_SERVICE_PROVIDER=protonvpn
	      - VPN_TYPE=wireguard
	      - WIREGUARD_PRIVATE_KEY=${PROTON_WIREGUARD_PRIVATE_KEY}
	      - SERVER_COUNTRIES=${VPN_SERVER_COUNTRIES}
	      - PORT_FORWARD_ONLY=on
	      - VPN_PORT_FORWARDING=on
	      - VPN_PORT_FORWARDING_PROVIDER=protonvpn
	      - VPN_PORT_FORWARDING_STATUS_FILE=/gluetun/forwarded_port
	      - 'HTTP_CONTROL_SERVER_AUTH_DEFAULT_ROLE={"auth":"apikey","apikey":"${GLUETUN_API_KEY}"}'
	      - FIREWALL_OUTBOUND_SUBNETS=172.20.0.0/24
	      - UPDATER_PERIOD=0
	    healthcheck:
	      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:9999 >/dev/null || exit 1"]
	      interval: 60s
	      timeout: 15s
	      retries: 3
	      start_period: 180s
	    deploy:
	      resources:
	        limits:
	          memory: 256M
	    logging:
	      driver: json-file
	      options:
	        max-size: "10m"
	        max-file: "3"
	networks:
	  ...
	```

3. Check the Docker Compose configuration:

	```bash
	cd /opt/mediaserver
	docker compose config --quiet && echo "OK"
	```

	* [ ] The command should return `OK`

<br>

### IV.&ensp;First startup

1. Start the container:

	```bash
	cd /opt/mediaserver
	docker compose up -d gluetun
	docker compose logs -f gluetun
	```

	* [ ] The logs should show a public IP address from your VPN provider, a `port forwarded is <PORT>` line, and no error (sometimes there is an issue with the selected server, try `docker compose restart gluetun` before debugging further)

2. Check that Docker sees the state correctly:

	```bash
	docker inspect --format '{{.State.Health.Status}}' gluetun
	```

	* [ ] The command should return `healthy`

3. Check the Docker status:

	```bash
	docker compose ps gluetun
	```

	* [ ] The `gluetun` container should have a `STATUS` of `Up`

<br>

### V.&ensp;Check your IP

1. Check the **[Gluetun](https://github.com/qdm12/gluetun)** IP:

	```bash
	curl -s -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" http://127.0.0.1:8000/v1/publicip/ip
	```

	* [ ] The IP in `public_ip` should be different from your real IP

2. Make sure authentication is enabled:

	```bash
	curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/v1/publicip/ip
	curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/v1/portforward
	```

	* [ ] Both commands should return `401`

<br>

### VI.&ensp;Check the port forwarding

1. Check the forwarded port:

	```bash
	curl -s -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" http://127.0.0.1:8000/v1/portforward
	```

	* [ ] The command should return a JSON with a non-empty non-zero `port` field (save it for a later check)

	<br>

	```bash
	cat /opt/mediaserver/config/gluetun/forwarded_port
	```

	* [ ] The command should return the same port as the previous command

<br>

### VII.&ensp;Check that other containers are using the VPN

1. Check the IP of another container using the VPN:

	```bash
	docker run --rm --network=container:gluetun alpine sh -c "wget -qO- https://ifconfig.me/ip; echo"
	```

	* [ ] The command should return the **[Gluetun](https://github.com/qdm12/gluetun)** IP, not your real IP

2. Check the DNS:

	```bash
	docker run --rm --network=container:gluetun alpine sh -c "nslookup github.com | grep -A1 'Name:'"
	```

	* [ ] The command should return a non-empty `Address` field

	<br>

	```bash
	docker run --rm --network=container:gluetun alpine sh -c "cat /etc/resolv.conf"
	```

	* [ ] The first line should be `nameserver 127.0.0.1`

3. Check for IPv6 leaks:

	```bash
	docker run --rm --network=container:gluetun alpine sh -c "wget -T 8 -qO- -6 https://ifconfig.co || echo NO-IPV6"
	```

	* [ ] You should see `NO-IPV6`

<br>

### VIII.&ensp;Check the kill switch

1. Stop the VPN:

	```bash
	curl -s -X PUT -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" -d '{"status":"stopped"}' http://127.0.0.1:8000/v1/vpn/status
	sleep 5 && curl -s -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" http://127.0.0.1:8000/v1/vpn/status
	```

	* [ ] The last command should return `{"status":"stopped"}`

2. Check the kill switch:

	```bash
	docker run --rm --network=container:gluetun alpine sh -c "wget -T 8 -qO- https://ifconfig.me/ip || echo BLOCKED"
	```

	* [ ] The command should return `BLOCKED`

3. Restart the VPN:

	```bash
	curl -s -X PUT -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" -d '{"status":"running"}' http://127.0.0.1:8000/v1/vpn/status
	sleep 20 && docker run --rm --network=container:gluetun alpine sh -c "wget -qO- https://ifconfig.me/ip; echo"
	```

	* [ ] The last command should return an IP address (not `BLOCKED` and not your real IP), save it for later

4. Wait for the port forwarding to be re-established:

	```bash
	for i in $(seq 1 30); do
	  PORT=$(curl -s -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" http://127.0.0.1:8000/v1/portforward | grep -oE '"port":[0-9]+' | cut -d: -f2)
	  echo "$(date +%H:%M:%S) port=$PORT"
	  [ "$PORT" != "0" ] && break
	  sleep 15
	done
	```

	* [ ] A non-zero port should be printed (can take a few minutes), and it should be different from the port of section **[VI.](#vicheck-the-port-forwarding)**, save it for later

	<br>

	```bash
	cat /opt/mediaserver/config/gluetun/forwarded_port
	```

	* [ ] The command should return the same port as the previous command

<br>

### IX.&ensp;Check the port forwarding from outside

1. Check the port from outside:

	```bash
	docker exec -it gluetun /bin/sh
	```

	In it:

	```bash
	wget -qO port-checker https://github.com/qdm12/port-checker/releases/download/v0.4.0/port-checker_0.4.0_linux_amd64
	chmod +x port-checker
	./port-checker --listening-address=":<YOUR_GLUETUN_PORT>"
	```

2. In a browser, go to `http://<YOUR_GLUETUN_IP>:<YOUR_GLUETUN_PORT>`:

	* [ ] It should show a page with your **[Gluetun](https://github.com/qdm12/gluetun)** port as `Listening address` and a private IP address like `10.2.0.x` as `Client address`, NOT your real IP address

	* [ ] A new log should be printed in the terminal with the same IP as the `Client address`

	(`Ctrl + C` and `exit` to quit)

<br>

### X.&ensp;Final verification

1. Reboot the local computer, then:

	```bash
	docker inspect -f '{{.State.Health.Status}}' gluetun
	```

	* [ ] The command should return `healthy`

	<br>

	```bash
	curl -s -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" http://127.0.0.1:8000/v1/portforward
	```

	* [ ] The command should show a non-zero port, most likely different from the previous ones

	<br>

	```bash
	docker logs gluetun 2>&1 | grep -ci "port forward"
	```

	* [ ] The command should return a non-zero count

2. Check for secret leaks:

	```bash
	docker logs gluetun 2>&1 | grep -c -F "<YOUR_PROTON_VPN_PRIVATE_KEY>"
	```

	* [ ] The command should return `0`

	<br>

	```bash
	docker logs gluetun 2>&1 | grep -c -F "$(grep '^GLUETUN_API_KEY=' /opt/mediaserver/.env | cut -d= -f2-)"
	```

	* [ ] The command should return `0`

<br>

### [Part 9: qBittorrent](./09_qbittorrent.md)
