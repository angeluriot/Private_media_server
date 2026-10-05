# Part 10: [Prowlarr](https://prowlarr.com/) and [Byparr](https://github.com/ThePhaseless/Byparr/)

### I.&ensp;Update the Docker Compose

1. On your local computer, open `/opt/mediaserver/docker-compose.yml` and add the following content:

	```yaml
	services:
	  jellyfin:
	    ...
	  gluetun:
	    ...
	  qbittorrent:
	    ...
	  prowlarr:
	    image: lscr.io/linuxserver/prowlarr:version-2.5.2.5491
	    container_name: prowlarr
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
	    volumes:
	      - /opt/mediaserver/config/prowlarr:/config
	    healthcheck:
	      test: ["CMD-SHELL", "wget -q --spider http://127.0.0.1:9696/ping || exit 1"]
	      interval: 60s
	      timeout: 10s
	      retries: 3
	      start_period: 60s
	    deploy:
	      resources:
	        limits:
	          memory: 512M
	    logging:
	      driver: json-file
	      options:
	        max-size: "10m"
	        max-file: "3"
	  byparr:
	    image: ghcr.io/thephaseless/byparr:3.0
	    container_name: byparr
	    restart: unless-stopped
	    network_mode: "service:gluetun"
	    depends_on:
	      gluetun:
	        condition: service_healthy
	        restart: true
	    shm_size: 512mb
	    environment:
	      - TZ=${TZ}
	      - LOG_LEVEL=info
	    deploy:
	      resources:
	        limits:
	          memory: 2G
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
	docker compose ps
	```

	* [ ] The last command should show all the containers with `Up` and `(healthy)`

	<br>

	```bash
	sleep 30 && docker inspect -f '{{.Name}} {{.State.Health.Status}}' gluetun qbittorrent prowlarr
	```

	* [ ] The three lines should be `healthy`

2. Check **[Byparr](https://github.com/ThePhaseless/Byparr/)**:

	```bash
	docker logs byparr | tail -20
	curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8191/docs
	```

	* [ ] Both commands should show `200`

	<br>

	```bash
	ls -ln /opt/mediaserver/config/prowlarr
	```

	* [ ] Every line should show `1000` `1000` (or your user)

<br>

### III.&ensp;Check that [Byparr](https://github.com/ThePhaseless/Byparr/) is working correctly

1. Check the IP address used by **[Byparr](https://github.com/ThePhaseless/Byparr/)**:

	```bash
	curl -s -X POST http://127.0.0.1:8191/v1 -H "Content-Type: application/json" -d '{"cmd":"request.get","url":"https://ifconfig.me/ip","maxTimeout":60000}' | grep -oE '[0-9]{1,3}(\.[0-9]{1,3}){3}'
	```

	* [ ] The command should return an IP address different from your public IP

2. Test with a real **[Cloudflare](https://www.cloudflare.com/)** protection:

	```bash
	curl -s -X POST http://127.0.0.1:8191/v1 -H "Content-Type: application/json" -d '{"cmd":"request.get","url":"https://nowsecure.nl","maxTimeout":60000}' | head -c 300
	```

	* [ ] The command should return a JSON with `"status":"ok"`

3. Check the kill switch:

	```bash
	curl -s -X PUT -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" -d '{"status":"stopped"}' http://127.0.0.1:8000/v1/vpn/status
	sleep 10 && curl -s -X POST http://127.0.0.1:8191/v1 -H "Content-Type: application/json" -d '{"cmd":"request.get","url":"https://ifconfig.me/ip","maxTimeout":20000}' | head -c 200
	```

	* [ ] The last command should return an error, not an IP address

	<br>

	```bash
	curl -s -X PUT -H "X-API-Key: <YOUR_GLUETUN_API_KEY>" -d '{"status":"running"}' http://127.0.0.1:8000/v1/vpn/status
	```

<br>

### IV.&ensp;Configure [Prowlarr](https://prowlarr.com/)

1. Go to **[localhost:9696](http://localhost:9696)** (or `http://<YOUR_LAN_IP>:9696` if you are on another computer in the LAN):

	| Field | Value |
	| --- | --- |
	| Authentication Method | `Forms (Login Page)` |
	| Authentication Required | `Enabled` |
	| Username | `<A_NEW_USERNAME>` |
	| Password | `<A_NEW_PASSWORD>` |
	| Password Confirmation | `<THE_SAME_PASSWORD>` |

	Save

2. Go to `Settings → General` and save the "API Key" for later

3. Go to `Settings → Indexers → Indexer Proxies → + → FlareSolverr`:

	| Field | Value |
	| --- | --- |
	| Name | `Byparr` |
	| Tags | `byparr` |
	| Host | `http://127.0.0.1:8191` |

	* [ ] Clicking on "Test" should return "​✅"

	Save

4. Go to `Settings → Download Clients → + → qBittorrent`:

	| Field | Value |
	| --- | --- |
	| Name | `qBittorrent` |
	| Host | `127.0.0.1` |
	| Username | `<YOUR_QBITTORRENT_USERNAME>` |
	| Password | `<YOUR_QBITTORRENT_PASSWORD>` |

	* [ ] Clicking on "Test" should return "​✅"

	Save

5. Go to `Indexers` (not `Settings → Indexers`) and add the indexers you want to use, if you don't know which ones to choose, you can try the ones I personally used: **[Prowlarr indexers appendix](./appendices/prowlarr_indexers.md)**

<br>

### V.&ensp;Verifications

1. Check all indexers added in the previous step:

	```bash
	curl -s -X POST -H "X-Api-Key: <YOUR_PROWLARR_API_KEY>" http://127.0.0.1:9696/api/v1/indexer/testall
	```

	* [ ] The command should return a JSON array with all elements having `"isValid": true`

2. Test a real search:

	```bash
	curl -s -H "X-Api-Key: <YOUR_PROWLARR_API_KEY>" "http://127.0.0.1:9696/api/v1/search?query=big%20buck%20bunny&type=search" | head -n 50
	```

	* [ ] The command should return a non-empty JSON array with elements containing fields such as `title` and `seeders`

3. On the web interface, go to `Search`, search for something popular, order the results by "Peers" (descending), and click on the download button on the right of one of the first results

4. Open the **[qBittorrent](https://www.qbittorrent.org/)** web interface:

	* [ ] A new download should appear in the list with the category `prowlarr`

	You can then delete it

5. Search for leaks:

	```bash
	docker logs prowlarr 2>&1 | grep -c -F "<YOUR_PROWLARR_API_KEY>"
	docker logs byparr 2>&1 | grep -c -F "<YOUR_LOCAL_IP>"
	docker logs prowlarr 2>&1 | grep -c -F "<YOUR_LOCAL_IP>"
	```

	* [ ] All commands should return `0`

6. Reboot your local computer, then:

	```bash
	sleep 60 && docker inspect -f '{{.Name}} {{.State.Health.Status}}' gluetun qbittorrent prowlarr
	```

	* [ ] All containers should have a `healthy` status

	<br>

	```bash
	curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8191/docs
	```

	* [ ] The command should return `200`

	<br>

	```bash
	curl -s -X POST -H "X-Api-Key: <YOUR_PROWLARR_API_KEY>" http://127.0.0.1:9696/api/v1/indexer/testall
	```

	* [ ] The command should return a JSON array with all elements having `"isValid": true`

<br>

### [Part 11: Radarr and Sonarr](./11_radarr_and_sonarr.md)
