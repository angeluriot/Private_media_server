# Part 12: [Bazarr](https://www.bazarr.media/)

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
	    ...
	  byparr:
	    ...
	  radarr:
	    ...
	  sonarr:
	    ...
	  bazarr:
	    image: lscr.io/linuxserver/bazarr:version-v1.6.1
	    container_name: bazarr
	    restart: unless-stopped
	    depends_on:
	      radarr:
	        condition: service_healthy
	      sonarr:
	        condition: service_healthy
	    networks:
	      - media
	    ports:
	      - "6767:6767/tcp"
	    environment:
	      - PUID=${PUID}
	      - PGID=${PGID}
	      - TZ=${TZ}
	      - UMASK=002
	    volumes:
	      - /opt/mediaserver/config/bazarr:/config
	      - /data/media:/data/media
	    healthcheck:
	      test: ["CMD-SHELL", "wget -q --spider http://127.0.0.1:6767/ || exit 1"]
	      interval: 60s
	      timeout: 10s
	      retries: 3
	      start_period: 90s
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

2. Check the Docker Compose configuration:

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
	docker compose up -d bazarr
	sleep 90 && docker inspect -f '{{.Name}} {{.State.Health.Status}}' bazarr
	```

	* [ ] The last command should return `/bazarr healthy`

2. Check the configuration files:

	```bash
	ls -ln /opt/mediaserver/config/bazarr
	```

	* [ ] All files should be owned by `1000` `1000` (or your user)

3. Check the container permissions:

	```bash
	docker exec bazarr ls -ln /data/media/movies
	ls -ln /data/media/movies
	```

	* [ ] Both commands should list the same items, both should show them owned by `1000` `1000` (or your user) and have `drwxrwsr-x` permissions

<br>

### III.&ensp;Authentication

1. Go to **[localhost:6767](http://localhost:6767)** (or `http://<YOUR_LAN_IP>:6767` if you are on another computer in the LAN) then `Settings → Application → General`:

	| Field | Value |
	| --- | --- |
	| Authentication | `Form` |
	| Username | `<A_NEW_USERNAME>` |
	| Password | `<A_NEW_PASSWORD>` |

	Then copy and save the API key for later

	Save

2. Reload the page and log in

3. Add the API key in the `.env`:

	```bash
	sudo tee -a /opt/mediaserver/.env > /dev/null <<'EOF'
	BAZARR_API_KEY=<YOUR_BAZARR_API_KEY>
	EOF
	```

4. Check the API:

	```bash
	curl -s -H "X-API-KEY: <YOUR_BAZARR_API_KEY>" http://127.0.0.1:6767/api/system/status
	```

	* [ ] The command should return a JSON object with the system status, not an error

<br>

### IV.&ensp;[Sonarr](https://sonarr.tv/) and [Radarr](https://radarr.video/) configuration

1. In `Settings → Library`:

	| Field | **[Sonarr](https://sonarr.tv/)** | **[Radarr](https://radarr.video/)** |
	| --- | --- | --- |
	| Use Sonarr/Radarr | `Enabled` | `Enabled` |
	| Address | `sonarr` | `radarr` |
	| Port | `8989` | `7878` |
	| API Key | `<YOUR_SONARR_API_KEY>` | `<YOUR_RADARR_API_KEY>` |
	| SSL | ❌ | ❌ |
	| Download Only Monitored | ✅ | ✅ |

	* [ ] The "Test" button should show a version and not an error

	Save

2. Restart **[Bazarr](https://www.bazarr.media/)**:

	```bash
	cd /opt/mediaserver
	docker compose restart bazarr
	```

3. Wait a few seconds, then in the web interface go to `Series`/`Movies`

	* [ ] You should see the content you already downloaded (anime are listed in `Series` with the other series)

4. In `Settings → Application → Scheduler`:

	| Field | Value |
	| --- | --- |
	| Sync with Sonarr | `15 Minutes` |
	| Sync with Radarr | `15 Minutes` |

	Save

<br>

### V.&ensp;Languages and profiles

1. On **[Bazarr](https://www.bazarr.media/)**, you can configure the languages and profiles for the subtitles, there is no standard configuration that fits everyone, but if you don't know where to start, you can check what I personally used: **[Bazarr languages and profiles appendix](./appendices/bazarr_languages_and_profiles.md)**

<br>

### VI.&ensp;Providers

1. Go to `Settings → Providers → Subtitles → +` and add the providers you want to use, if you don't know which ones to choose, you can try the ones I personally used: **[Bazarr providers appendix](./appendices/bazarr_providers.md)**

	* [ ] In `System → Providers`, they should all be listed with Status "Good"

<br>

### VII.&ensp;Subtitles configuration

1. In `Settings → Subtitles → Files`:

	| Field | Value |
	| --- | --- |
	| Hearing-impaired subtitles extension | `.sdh (Subtitles for the Deaf or Hard-of-Hearing)` |
	| Ignore Embedded PGS Subtitles | ✅ |
	| Ignore Embedded VobSub Subtitles | ✅ |

	Save

2. In `Settings → Subtitles → Processing`:

	| Field | Value |
	| --- | --- |
	| OCR Fixes | ✅ |
	| Common Fixes | ✅ |

	Save

<br>

### VIII.&ensp;[Jellyfin](https://jellyfin.org/) configuration

1. In **[Jellyfin](https://jellyfin.org/)** (`tv.<YOUR_DOMAIN>`), go to `Dashboard → API Keys → +` and create one `Bazarr`

2. In **[Bazarr](https://www.bazarr.media/)**, go to `Settings → Integrations → Jellyfin`:

	| Field | Value |
	| --- | --- |
	| Use Jellyfin Media Server | `Enabled` |
	| Server URL | `http://jellyfin:8096` |
	| API Key | `<YOUR_JELLYFIN_BAZARR_API_KEY>` |
	| How to notify Jellyfin after subtitle changes | `Immediate` |

	Click on "Test Connection"

	Movie Library:

	| Field | Value |
	| --- | --- |
	| Library Name | `Movies` |
	| Refresh movie metadata after downloading subtitles | ✅ |

	Series Library:

	| Field | Value |
	| --- | --- |
	| Library Name | `TV Shows` and `Anime` (select both) |
	| Refresh series metadata after downloading subtitles | ✅ |

	Save

	> ⚠️ There is currently a bug in **[Bazarr](https://www.bazarr.media/)** that prevents **[Jellyfin](https://jellyfin.org/)** from retrieving metadata when "Refresh series metadata after downloading subtitles" is enabled, if you see media not imported correctly, try disabling this option until an update is released

<br>

### IX.&ensp;Translation

1. Go to **[Google AI Studio](https://aistudio.google.com/apikey)**, log in with your Google account, click on `Create API key` and copy it

2. In **[Bazarr](https://www.bazarr.media/)**, go to `Settings → Providers → Translation`:

	| Field | Value |
	| --- | --- |
	| Translator | `Gemini` |
	| Gemini model | `gemini-3.5-flash-lite` (or `gemini-3.8-flash` for better translations) |
	| Gemini batch size | `300` |
	| Gemini API keys | `<YOUR_GEMINI_API_KEY>` |
	| Add translation info at the beginning | ✅ |

	Save

3. To translate a subtitle, open a movie or an episode, click on an existing subtitle then `Translate...`, select the target language and click on `Start`

<br>

### [Part 13: Seerr](./13_seerr.md)
