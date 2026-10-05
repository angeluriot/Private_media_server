# Part 13: [Seerr](https://seerr.dev/)

### I.&ensp;Update the Docker Compose

1. On your local computer, create the **[Seerr](https://seerr.dev/)** config directory:

	```bash
	sudo mkdir -p /opt/mediaserver/config/seerr
	sudo chown -R 1000:1000 /opt/mediaserver/config/seerr
	```

2. Open `/opt/mediaserver/docker-compose.yml` and add the following content:

	```yaml
	services:
	  jellyfin:
	    ...
	  gluetun:
	    ...
	  qbittorrent:
	    ...
	  prowlarr:
	    ...
	  byparr:
	    ...
	  radarr:
	    ...
	  sonarr:
	    ...
	  bazarr:
	    ...
	  seerr:
	    image: ghcr.io/seerr-team/seerr:v3.4
	    container_name: seerr
	    init: true
	    restart: unless-stopped
	    user: "${PUID}:${PGID}"
	    depends_on:
	      radarr:
	        condition: service_healthy
	      sonarr:
	        condition: service_healthy
	    networks:
	      - media
	    ports:
	      - "5055:5055/tcp"
	    environment:
	      - TZ=${TZ}
	      - LOG_LEVEL=info
	      - PORT=5055
	    volumes:
	      - /opt/mediaserver/config/seerr:/app/config
	    healthcheck:
	      test: ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://127.0.0.1:5055/api/v1/settings/public || exit 1"]
	      interval: 60s
	      timeout: 10s
	      retries: 3
	      start_period: 60s
	    security_opt:
	      - no-new-privileges
	    cap_drop:
	      - ALL
	    deploy:
	      resources:
	        limits:
	          memory: 512M
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

### II.&ensp;First startup

1. Start the container:

	```bash
	cd /opt/mediaserver
	docker compose up -d seerr
	docker compose logs -f seerr
	```

	* [ ] The last command should end with a line `Server ready on port 5055` and no errors

2. Check the health:

	```bash
	sleep 60 && docker inspect -f '{{.Name}} {{.State.Health.Status}}' seerr
	```

	* [ ] The command should return `/seerr healthy`

3. Check the permissions:

	```bash
	docker exec seerr id
	```

	* [ ] The command should return `1000` for `uid` and `gid`

	<br>

	```bash
	ls -ln /opt/mediaserver/config/seerr
	```

	* [ ] All files should be owned by `1000` `1000` (or your user)

4. Check the API:

	```bash
	curl -s http://127.0.0.1:5055/api/v1/status
	```

	* [ ] The command should return a JSON object with the status of **[Seerr](https://seerr.dev/)**

<br>

### III.&ensp;Login

1. Go to **[localhost:5055](http://localhost:5055)** (or `http://<YOUR_LAN_IP>:5055` if you are on another computer in the LAN) and select "Configure Jellyfin":

	| Field | Value |
	| --- | --- |
	| Jellyfin URL | `jellyfin` |
	| Email Address | `<YOUR_EMAIL_ADDRESS>` |
	| Username | `<YOUR_JELLYFIN_USERNAME>` |
	| Password | `<YOUR_JELLYFIN_PASSWORD>` |

	Click on "Sign In"

2. On the next screen, click on "Sync Libraries", select `Movies`, `TV Shows` and `Anime`, and click on "Start Scan":

	| Field | Value |
	| --- | --- |
	| External URL | `https://tv.<YOUR_DOMAIN>` |

	Click on "Continue"

3. On the next screen, click on "Add Radarr Server":

	| Field | Value |
	| --- | --- |
	| Default Server | ✅ |
	| Server Name | `Radarr` |
	| Hostname or IP Address | `radarr` |
	| Port | `7878` |
	| API Key | `<YOUR_RADARR_API_KEY>` |

	Then click on "Test"

	| Field | Value |
	| --- | --- |
	| Quality Profile | `<YOUR_MAIN_QUALITY_PROFILE>` |
	| Root Folder | `/data/media/movies/` |
	| Minimum Availability | `Released` |
	| External URL | `http://localhost:7878` |
	| Enable Scan | ✅ |
	| Enable Automatic Search | ✅ |

	Click on "Add Server"

4. Click on "Add Sonarr Server":

	| Field | Value |
	| --- | --- |
	| Default Server | ✅ |
	| Server Name | `Sonarr` |
	| Hostname or IP Address | `sonarr` |
	| Port | `8989` |
	| API Key | `<YOUR_SONARR_API_KEY>` |

	Then click on "Test"

	| Field | Value |
	| --- | --- |
	| Series Type | `Standard` |
	| Quality Profile | `<YOUR_MAIN_QUALITY_PROFILE>` |
	| Root Folder | `/data/media/tv/` |
	| Anime Series Type | `Anime` |
	| Anime Quality Profile | `<YOUR_MAIN_OR_ANIME_QUALITY_PROFILE>` |
	| Anime Root Folder | `/data/media/anime/` |
	| Anime Tags | `anime` |
	| Season Folders | ✅ |
	| External URL | `http://localhost:8989` |
	| Enable Scan | ✅ |
	| Enable Automatic Search | ✅ |

	Click on "Add Server", then "Finish Setup"

<br>

### IV.&ensp;Configuration

1. In `Settings → General`:

	| Field | Value |
	| --- | --- |
	| Application Title | `<A_NAME_FOR_YOUR_MEDIA_SERVER>` |
	| Application URL | `https://request.<YOUR_DOMAIN>` |
	| Display Language | `<YOUR_LANGUAGE>` |
	| Streaming Region | `<YOUR_COUNTRY>` |
	| Allow Special Episodes Requests | ✅ |

	Save

2. In `Settings → Users`:

	| Field | Value |
	| --- | --- |
	| Enable Local Sign-In | ❌ |
	| Global Movie Request Limit | `20` movies per `10` days (or any limit you prefer) |
	| Global Series Request Limit | `5` seasons per `10` days (or any limit you prefer) |
	| Advanced Requests | ✅ |
	| View Recently Added | ✅ |
	| Auto-Approve | ✅ |

	Save

3. In `Settings → Metadata Providers`:

	| Field | Value |
	| --- | --- |
	| Series metadata provider | `TheTVDB` |
	| Anime metadata provider | `TheTVDB` |

	Save

<br>

### V.&ensp;Network

1. Go to `Settings → General` and copy "API Key"

2. Add the API key to the `.env` file:

	```bash
	sudo tee -a /opt/mediaserver/.env > /dev/null <<'EOF'
	SEERR_API_KEY=<YOUR_SEERR_API_KEY>
	EOF
	```

3. In `Settings → Network`:

	| Field | Value |
	| --- | --- |
	| Enable Proxy Support | ✅ |

	Save

4. Restart **[Seerr](https://seerr.dev/)**:

	```bash
	cd /opt/mediaserver
	docker compose restart seerr
	```

5. Check the changes:

	```bash
	curl -s -H "X-Api-Key: <YOUR_SEERR_API_KEY>" http://127.0.0.1:5055/api/v1/settings/main | grep -oE '"applicationUrl":"[^"]*"'
	curl -s -H "X-Api-Key: <YOUR_SEERR_API_KEY>" http://127.0.0.1:5055/api/v1/settings/network | grep -oE '"trustProxy":(true|false)|"csrfProtection":(true|false)'
	```

	* [ ] The first command should return `https://request.<YOUR_DOMAIN>` and the second should return `"csrfProtection":false` and `"trustProxy":true`

<br>

### VI.&ensp;Connect [Seerr](https://seerr.dev/) to the VPS

1. On the VPS, open `/etc/caddy/Caddyfile` and add the following content:

	```caddyfile
	{ ... }

	*.<YOUR_DOMAIN> {
		tls { ... }
		log { ... }
		header { ... }
		handle /robots.txt { ... }

		@tv host tv.<YOUR_DOMAIN>
		handle @tv { ... }

		@request host request.<YOUR_DOMAIN>
		handle @request {
			reverse_proxy 10.10.0.2:5055
		}

		handle { ... }
	}
	```

2. Check the syntax:

	```bash
	sudo caddy fmt --overwrite /etc/caddy/Caddyfile
	sudo caddy adapt --config /etc/caddy/Caddyfile > /dev/null && echo "Syntax OK"
	```

	* [ ] The last command should return `Syntax OK`

3. Reload **[Caddy](https://caddyserver.com/)**:

	```bash
	sudo systemctl reload caddy
	sudo systemctl status caddy --no-pager
	```

	* [ ] The last command should show `active (running)` and no errors

4. In your browser, go to `https://request.<YOUR_DOMAIN>` and log in with your **[Jellyfin](https://jellyfin.org/)** credentials

	* [ ] Everything should be working correctly

5. In private browsing, go to `https://request.<YOUR_DOMAIN>` and try to log in with a wrong password

6. On the local computer:

	```bash
	docker logs seerr 2>&1 | grep -iE "logged in|sign-in" | tail -3
	```

	* [ ] The last line should be a warning with your IP address (not `172.20.0.1`)

<br>

### VII.&ensp;Brute-force protection

1. On your local computer, create the acquisition file:

	```bash
	sudo tee /etc/crowdsec/acquis.d/seerr.yaml > /dev/null <<'EOF'
	source: docker
	container_name:
	  - seerr
	labels:
	  type: seerr
	EOF
	```

2. Create the parser:

	```bash
	sudo tee /etc/crowdsec/parsers/s01-parse/seerr-logs.yaml > /dev/null <<'EOF'
	onsuccess: next_stage
	name: custom/seerr-logs
	description: "Parse Seerr failed sign-in attempts"
	filter: "evt.Parsed.program == 'seerr'"
	nodes:
	  - grok:
	      pattern: '^%{TIMESTAMP_ISO8601:timestamp} \[%{WORD:level}\]\[Auth\]: Failed (sign-in|login) attempt.*"ip":"(::ffff:)?%{IP:source_ip}","email":"%{DATA:username}"'
	      apply_on: message
	      statics:
	        - meta: log_type
	          value: seerr_failed_auth
	statics:
	  - meta: service
	    value: seerr
	  - meta: source_ip
	    expression: "evt.Parsed.source_ip"
	  - meta: user
	    expression: "evt.Parsed.username"
	  - target: evt.StrTime
	    expression: "evt.Parsed.timestamp"
	EOF
	```

3. Create the scenario:

	```bash
	sudo tee /etc/crowdsec/scenarios/seerr-bf.yaml > /dev/null <<'EOF'
	type: leaky
	name: custom/seerr-bf
	description: "Detect Seerr sign-in brute force"
	filter: "evt.Meta.log_type == 'seerr_failed_auth'"
	groupby: evt.Meta.source_ip
	leakspeed: "2m"
	capacity: 9
	blackhole: 2m
	labels:
	  service: seerr
	  confidence: 3
	  spoofable: 0
	  classification:
	    - attack.T1110
	  label: "Seerr brute force"
	  behavior: "http:bruteforce"
	  remediation: true
	EOF
	```

4. Restart **[CrowdSec](https://www.crowdsec.net/)**:

	```bash
	sudo systemctl restart crowdsec
	sudo systemctl status crowdsec --no-pager
	```

	* [ ] The last command should show `active (running)` and no errors

	<br>

	```bash
	sudo cscli parsers list | grep seerr
	sudo cscli scenarios list | grep seerr
	```

	* [ ] `custom/seerr-logs` and `custom/seerr-bf` should both be listed as `enabled,local`

5. In private browsing, go to `https://request.<YOUR_DOMAIN>` and try to log in with a wrong password

6. Check that the logs are read:

	```bash
	sudo cscli metrics
	```

	* [ ] In the `Acquisition Metrics` section, a `docker:seerr` line should appear with a non-zero number of lines read

7. Check the parsing:

	```bash
	sudo cscli explain --type seerr --log '2026-01-01T00:00:00.000Z [warn][Auth]: Failed sign-in attempt from user with incorrect Jellyfin credentials {"account":{"ip":"192.0.2.1","email":"test","password":"__REDACTED__"}}'
	```

	* [ ] `custom/seerr-bf` should be 🟢 at the bottom, in the scenarios section

8. Using your phone on 5G, go to `https://request.<YOUR_DOMAIN>` and try to log in with a wrong password repeatedly

	* [ ] After a few tries, the website should freeze and stop working

9. On the VPS:

	```bash
	sudo cscli alerts list
	```

	* [ ] A line with your mobile operator and your phone's IP address should be listed

	<br>

	```bash
	sudo cscli decisions list
	```

	* [ ] A line with your mobile operator and your phone's IP address should be listed with the action `ban`

	<br>

	```bash
	sudo cscli decisions delete --all
	```

	* [ ] After a few minutes, you should have access to the website again with your phone

<br>

### VIII.&ensp;End-to-end test

1. On **[Jellyfin](https://jellyfin.org/)**, in `Dashboard → Users → +`, choose a name and a password and check "Enable access to all libraries", on the next screen, save

2. On **[Seerr](https://seerr.dev/)**, try to log in with the newly created **[Jellyfin](https://jellyfin.org/)** user credentials

	* [ ] It should work

3. Try to request something popular (you should be able to choose the profile)

	* [ ] The request should be transmitted to **[Radarr](https://radarr.video/)** / **[Sonarr](https://sonarr.tv/)**

	* [ ] The download should be visible in **[qBittorrent](https://www.qbittorrent.org/)**

	* [ ] When the download is complete, the content should be available in **[Jellyfin](https://jellyfin.org/)**

	* [ ] The content should also be visible in **[Bazarr](https://www.bazarr.media/)**

4. Try to request a popular anime series

	* [ ] In **[Sonarr](https://sonarr.tv/)**, the series should have `Anime` as "Series Type", `/data/media/anime/` as "Root Folder" and the `anime` tag

	* [ ] The download should be visible in **[qBittorrent](https://www.qbittorrent.org/)** under the `anime` category

	* [ ] When the download is complete, the content should be available in the `Anime` library of **[Jellyfin](https://jellyfin.org/)** (not in `TV Shows`)

<br>

### [Part 14: Users](./14_users.md)
