# Part 9: [qBittorrent](https://www.qbittorrent.org/)

### I.&ensp;Update the Docker Compose

1. On your local computer, open `/opt/mediaserver/docker-compose.yml` and add the following content:

	```yaml
	services:
	  jellyfin:
	    ...
	  gluetun:
	    ...
	  qbittorrent:
	    image: lscr.io/linuxserver/qbittorrent:5.2.3
	    container_name: qbittorrent
	    restart: unless-stopped
	    network_mode: "service:gluetun"
	    depends_on:
	      gluetun:
	        condition: service_healthy
	        restart: true
	    environment:
	      - PUID=${PUID}
	      - PGID=${PGID}
	      - TZ=${TZ}
	      - UMASK=002
	      - WEBUI_PORT=8080
	    volumes:
	      - /opt/mediaserver/config/qbittorrent:/config
	      - /data:/data
	    healthcheck:
	      test: ["CMD-SHELL", "wget -q --spider http://127.0.0.1:8080/ || exit 1"]
	      interval: 60s
	      timeout: 10s
	      retries: 3
	      start_period: 60s
	    deploy:
	      resources:
	        limits:
	          memory: 1G
	    logging:
	      driver: json-file
	      options:
	        max-size: "10m"
	        max-file: "3"
	networks:
	  ...
	```

2. Check the Docker Compose configuration:

	```bash
	cd /opt/mediaserver
	docker compose config --quiet && echo "OK"
	```

	* [ ] The command should return `OK`

<br>

### II.&ensp;First startup

1. Start the containers:

	```bash
	cd /opt/mediaserver
	docker compose up -d
	docker compose logs qbittorrent | grep -i "temporary password"
	```

	* [ ] The last command should return the temporary password for the **[qBittorrent](https://www.qbittorrent.org/)** Web UI, save it for later

2. Check the health of the containers:

	```bash
	docker inspect -f '{{.State.Health.Status}}' gluetun qbittorrent
	```

	* [ ] The command should return `healthy` two times

3. Check the permissions:

	```bash
	docker top qbittorrent | grep qbittorrent-nox
	```

	* [ ] The line should start with `1000` (or your user)

	<br>

	```bash
	docker exec qbittorrent ps -eo user,uid,comm | grep -i qbittorrent
	```

	* [ ] The line should contain `1000` (or your user)

	<br>

	```bash
	ls -ln /opt/mediaserver/config/qbittorrent
	```

	* [ ] The line should contain `1000` `1000` (or your user)

<br>

### III.&ensp;Configure the Web UI

1. Go to **[localhost:8080](http://localhost:8080)** (or `http://<YOUR_LAN_IP>:8080` if you are on another computer in the LAN) and log in using `admin` and the temporary password

2. Go to `Options → WebUI`:

	| Field | Value |
	| --- | --- |
	| Username | `<A_NEW_USERNAME>` |
	| Password | `<A_NEW_PASSWORD>` |
	| Bypass authentication for clients on localhost | ​✅ |

	Save

<br>

### IV.&ensp;Connect [qBittorrent](https://www.qbittorrent.org/) to [Gluetun](https://github.com/qdm12/gluetun)

1. Open `/opt/mediaserver/docker-compose.yml` and add the following content:

	```yaml
	services:
	  jellyfin:
	    ...
	  gluetun:
	    ...
	    environment:
	      ...
	      - >-
	        VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -qO-
	        --retry-connrefused --waitretry=2 --timeout=15
	        --post-data="json={\"listen_port\":{{PORTS}},\"current_network_interface\":\"{{VPN_INTERFACE}}\",\"random_port\":false,\"upnp\":false}"
	        http://127.0.0.1:8080/api/v2/app/setPreferences'
	      - >-
	        VPN_PORT_FORWARDING_DOWN_COMMAND=/bin/sh -c 'wget -qO-
	        --post-data="json={\"listen_port\":0}"
	        http://127.0.0.1:8080/api/v2/app/setPreferences'
	    ...
	  qbittorrent:
	    ...
	networks:
	  ...
	```

2. Check the Docker Compose configuration:

	```bash
	cd /opt/mediaserver
	docker compose config --quiet && echo "OK"
	```

	* [ ] The command should return `OK`

3. Restart the Docker containers to apply the changes:

	```bash
	cd /opt/mediaserver
	docker compose up -d --force-recreate gluetun qbittorrent
	docker compose logs -f gluetun
	```

	* [ ] The last command should not return any error

	<br>

	```bash
	sleep 20 && docker exec qbittorrent curl -s http://127.0.0.1:8080/api/v2/app/preferences | grep -oE '"listen_port":[0-9]+'
	cat /opt/mediaserver/config/gluetun/forwarded_port
	```

	* [ ] Both commands should return the same port

<br>

### V.&ensp;Configure [qBittorrent](https://www.qbittorrent.org/)

1. Go to the **[qBittorrent](https://www.qbittorrent.org/)** web interface, then `Options → Downloads`:

	| Field | Value |
	| --- | --- |
	| Append .!qB extension to incomplete files | ​✅ |
	| Default Torrent Management Mode | `Automatic` |
	| When Default Save Path changed | `Relocate affected torrents` |
	| When Category Save Path changed | `Relocate affected torrents` |
	| Default Save Path | `/data/torrents` |
	| Keep incomplete torrents in | ​✅ `/data/torrents/incomplete` |

	Save

2. Right-click on the `Category` column and select `New Category` to add the following categories:

	* `movies`: `/data/torrents/movies`
	* `tv`: `/data/torrents/tv`
	* `anime`: `/data/torrents/anime`

3. In `Options → Connection`:

	| Field | Value |
	| --- | --- |
	| Global maximum number of upload slots | ​✅ `100` |
	| Maximum number of upload slots per torrent | ​✅ `10` |

	Save

4. In `Options → BitTorrent`:

	| Field | Value |
	| --- | --- |
	| Enable Local Peer Discovery to find more peers | ❌ |
	| Torrent Queueing | ​✅ |
	| Maximum active downloads | `10` |
	| Maximum active uploads | `-1` |
	| Maximum active torrents | `-1` |
	| Do not count slow torrents in these limits | ​✅ |
	| Automatically append trackers from URL to new downloads | ​✅ `https://raw.githubusercontent.com/ngosang/trackerslist/master/trackers_best.txt` |

	Save

5. In `Options → Advanced`, set `Disk IO type (requires restart)` to `Simple pread/pwrite`

	Save

6. Restart **[qBittorrent](https://www.qbittorrent.org/)** to apply the changes:

	```bash
	cd /opt/mediaserver
	docker compose restart qbittorrent
	```

<br>

### VI.&ensp;Verifications

1. Check the IP address of the **[qBittorrent](https://www.qbittorrent.org/)** container:

	```bash
	docker exec qbittorrent curl -s https://ifconfig.me/ip; echo
	docker exec qbittorrent ip -o addr show tun0 | awk '{print $4}'
	```

	* [ ] The first command should return a different IP than yours and the second should return an IP starting with `10.2.0.`

2. Test a download:

	```bash
	docker exec qbittorrent curl -sf -F "urls=https://webtorrent.io/torrents/sintel.torrent" -F "category=movies" http://127.0.0.1:8080/api/v2/torrents/add
	```

	* [ ] In the web interface, the downloading file should appear under the `movies` category

	* [ ] When the download is complete, the file should have a status of `Seeding`

	<br>

	```bash
	ls -l /data/torrents/movies/
	```

	* [ ] The downloaded file should be listed with `1000` `1000` (or your user)

	<br>

	```bash
	find /data/torrents/movies -type d -exec stat -c '%U:%G %a %n' {} \;
	find /data/torrents/movies -type f -exec stat -c '%U:%G %a %n' {} \;
	```

	* [ ] The first command should list `2775` directories and the second `664` files

3. Check the kill switch:

	```bash
	curl -s -X PUT -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" -d '{"status":"stopped"}' http://127.0.0.1:8000/v1/vpn/status
	sleep 10 && docker exec qbittorrent curl -s --max-time 8 https://ifconfig.me/ip || echo BLOCKED
	```

	* [ ] The last command should return `BLOCKED`

	* [ ] In the web interface, the number of "Peers" should be constantly `0`

	<br>

	```bash
	curl -s -X PUT -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" -d '{"status":"running"}' http://127.0.0.1:8000/v1/vpn/status
	```

	* [ ] After a few minutes, the number of "Peers" should sometimes increase above `0`

	You can now delete the test download in the web interface (check deleting the associated files as well)

4. Reboot the local computer, then:

	```bash
	sleep 20 && docker inspect -f '{{.State.Health.Status}}' gluetun qbittorrent
	```

	* [ ] The command should return `healthy` two times

	<br>

	```bash
	docker exec qbittorrent curl -s http://127.0.0.1:8080/api/v2/app/preferences | grep -oE '"listen_port":[0-9]+'
	cat /opt/mediaserver/config/gluetun/forwarded_port
	```

	* [ ] Both commands should return the same port

<br>

### [Part 10: Prowlarr and Byparr](./10_prowlarr_and_byparr.md)
