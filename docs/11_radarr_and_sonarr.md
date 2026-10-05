# Part 11: [Radarr](https://radarr.video/) and [Sonarr](https://sonarr.tv/)

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
	    image: lscr.io/linuxserver/radarr:version-6.3.0.10514
	    container_name: radarr
	    restart: unless-stopped
	    depends_on:
	      gluetun:
	        condition: service_healthy
	    networks:
	      media:
	        ipv4_address: 172.20.0.20
	    ports:
	      - "7878:7878/tcp"
	    environment:
	      - PUID=${PUID}
	      - PGID=${PGID}
	      - TZ=${TZ}
	      - UMASK=002
	    volumes:
	      - /opt/mediaserver/config/radarr:/config
	      - /data:/data
	    healthcheck:
	      test: ["CMD-SHELL", "wget -q --spider http://127.0.0.1:7878/ping || exit 1"]
	      interval: 60s
	      timeout: 10s
	      retries: 3
	      start_period: 60s
	    deploy:
	      resources:
	        limits:
	          memory: 768M
	    logging:
	      driver: json-file
	      options:
	        max-size: "10m"
	        max-file: "3"
	  sonarr:
	    image: lscr.io/linuxserver/sonarr:version-4.0.19.2979
	    container_name: sonarr
	    restart: unless-stopped
	    depends_on:
	      gluetun:
	        condition: service_healthy
	    networks:
	      media:
	        ipv4_address: 172.20.0.21
	    ports:
	      - "8989:8989/tcp"
	    environment:
	      - PUID=${PUID}
	      - PGID=${PGID}
	      - TZ=${TZ}
	      - UMASK=002
	    volumes:
	      - /opt/mediaserver/config/sonarr:/config
	      - /data:/data
	    healthcheck:
	      test: ["CMD-SHELL", "wget -q --spider http://127.0.0.1:8989/ping || exit 1"]
	      interval: 60s
	      timeout: 10s
	      retries: 3
	      start_period: 60s
	    deploy:
	      resources:
	        limits:
	          memory: 768M
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
	docker compose up -d radarr sonarr
	sleep 60 && docker inspect -f '{{.Name}} {{.State.Health.Status}}' radarr sonarr
	```

	* [ ] Both `radarr` and `sonarr` should show `healthy` status

2. Check the containers' network:

	```bash
	docker inspect -f '{{.Name}} {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' radarr sonarr
	```

	* [ ] `radarr` should have IP `172.20.0.20` and `sonarr` should have IP `172.20.0.21`

	<br>

	```bash
	docker exec prowlarr wget -qO- --timeout=5 http://172.20.0.20:7878/ping
	docker exec prowlarr wget -qO- --timeout=5 http://172.20.0.21:8989/ping
	```

	* [ ] Both commands should return `{"status": "OK"}`

<br>

### III.&ensp;API keys

1. Retrieve the API keys:

	```bash
	docker exec radarr sed -n 's:.*<ApiKey>\(.*\)</ApiKey>.*:\1:p' /config/config.xml
	docker exec sonarr sed -n 's:.*<ApiKey>\(.*\)</ApiKey>.*:\1:p' /config/config.xml
	```

	Save them for later

2. Add both keys in the `.env` file:

	```bash
	sudo tee -a /opt/mediaserver/.env > /dev/null <<'EOF'
	RADARR_API_KEY=<YOUR_RADARR_API_KEY>
	SONARR_API_KEY=<YOUR_SONARR_API_KEY>
	EOF
	```

3. Check the permissions:

	```bash
	ls -ln /opt/mediaserver/config/radarr /opt/mediaserver/config/sonarr
	```

	* [ ] All files should have `1000` `1000` (or your user)

<br>

### IV.&ensp;[Radarr](https://radarr.video/) and [Sonarr](https://sonarr.tv/) configuration

1. Go to **[localhost:7878](http://localhost:7878)** (**[Radarr](https://radarr.video/)**) and **[localhost:8989](http://localhost:8989)** (**[Sonarr](https://sonarr.tv/)**) (or `http://<YOUR_LAN_IP>:7878` / `http://<YOUR_LAN_IP>:8989` if you are on another computer in the LAN):

	| Field | Value |
	| --- | --- |
	| Authentication Method | `Forms (Login Page)` |
	| Authentication Required | `Enabled` |
	| Username | `<A_NEW_USERNAME>` |
	| Password | `<A_NEW_PASSWORD>` |

	Save

2. Create a recycle bin folder for deleted movies:

	```bash
	mkdir -p /data/.recyclebin && chmod 775 /data/.recyclebin
	```

3. In the **[Radarr](https://radarr.video/)** web interface, go to `Settings → Media Management`, click on "Show Advanced", then:

	| Field | Value |
	| --- | --- |
	| Rename Movies | ✅ |
	| Replace Illegal Characters | ✅ |
	| Standard Movie Format | `{Movie CleanTitle} ({Release Year}) [tmdbid-{TmdbId}] {[Quality Full]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels]}{[Mediainfo VideoCodec]}{-Release Group}` |
	| Movie Folder Format | `{Movie CleanTitle} ({Release Year}) [tmdbid-{TmdbId}]` |
	| Minimum Free Space | `10000` |
	| Use Hardlinks instead of Copy | ✅ |
	| Import Extra Files | ✅ |
	| Import Extra Files | `srt,sub,idx,ass` |
	| Unmonitor Deleted Movies | ✅ |
	| Propers and Repacks | `Do Not Prefer` |
	| Analyze video files | ✅ |
	| Recycling Bin | `/data/.recyclebin/` |
	| Set Permissions | ❌ |

	Then click on "Add Root Folder" and set `/data/media/movies/`

	Save

4. In the **[Sonarr](https://sonarr.tv/)** web interface, go to `Settings → Media Management`, click on "Show Advanced", then:

	| Field | Value |
	| --- | --- |
	| Rename Episodes | ✅ |
	| Replace Illegal Characters | ✅ |
	| Standard Episode Format | `{Series TitleYear} - S{season:00}E{episode:00} - {Episode CleanTitle} {[Quality Full]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels]}{[Mediainfo VideoCodec]}{-Release Group}` |
	| Daily Episode Format | `{Series TitleYear} - {Air-Date} - {Episode CleanTitle} {[Quality Full]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels]}{[Mediainfo VideoCodec]}{-Release Group}` |
	| Anime Episode Format | `{Series TitleYear} - S{season:00}E{episode:00} - {absolute:000} - {Episode CleanTitle} {[Quality Full]}{[MediaInfo VideoDynamicRangeType]}[{MediaInfo VideoBitDepth}bit]{[Mediainfo VideoCodec]}[{Mediainfo AudioCodec} {Mediainfo AudioChannels}]{-Release Group}` |
	| Series Folder Format | `{Series TitleYear} [tvdbid-{TvdbId}]` |
	| Season Folder Format | `Season {season:00}` |
	| Minimum Free Space | `10000` |
	| Use Hardlinks instead of Copy | ✅ |
	| Import Extra Files | ✅ |
	| Import Extra Files | `srt,sub,idx,ass` |
	| Unmonitor Deleted Episodes | ✅ |
	| Propers and Repacks | `Do Not Prefer` |
	| Analyze video files | ✅ |
	| Recycling Bin | `/data/.recyclebin/` |
	| Set Permissions | ❌ |

	Then click on "Add Root Folder" two times (`/data/media/tv/` and `/data/media/anime/`)

	Save

5. Check that they can write to the respective media folders:

	```bash
	curl -s -H "X-Api-Key: <YOUR_RADARR_API_KEY>" http://127.0.0.1:7878/api/v3/rootfolder
	curl -s -H "X-Api-Key: <YOUR_SONARR_API_KEY>" http://127.0.0.1:8989/api/v3/rootfolder
	```

	* [ ] Both commands should return a JSON with `"accessible": true` and a non-null `"freeSpace"` (two of them for **[Sonarr](https://sonarr.tv/)**: `/data/media/tv/` and `/data/media/anime/`)

6. If you already have media in your media folders (**[Big Buck Bunny (2008)](https://en.wikipedia.org/wiki/Big_Buck_Bunny)** for example):

	Import them with `Library Import → /data/media/movies/`

	Go back to `Movies`, click on "Edit Movies", select them all, then click on "Rename Files" and "Organize"

	Click on "Edit Movies" again, select them all, click on "Edit", leave everything as "No Change" except the "Root Folder" which should be set to `/data/media/movies/`, then click on "Apply Changes" and "Yes, Move the Files"

	The folders and files should now be organized according to the format specified in the settings

	Do the same in **[Sonarr](https://sonarr.tv/)** with `/data/media/tv/` and `/data/media/anime/` if you already have series (choose `Anime` as "Series Type" for the ones in `/data/media/anime/`)

7. In **[Radarr](https://radarr.video/)** and **[Sonarr](https://sonarr.tv/)** (2 times), go to `Settings → Download Clients → + → qBittorrent`:

	| Field | **[Radarr](https://radarr.video/)** | **[Sonarr](https://sonarr.tv/)** 1 | **[Sonarr](https://sonarr.tv/)** 2 |
	| --- | --- | --- | --- |
	| Name | `qBittorrent` | `qBittorrent (tv)` | `qBittorrent (anime)` |
	| Host | `gluetun` | `gluetun` | `gluetun` |
	| Port | `8080` | `8080` | `8080` |
	| Username | `<YOUR_QBITTORRENT_USERNAME>` | `<YOUR_QBITTORRENT_USERNAME>` | `<YOUR_QBITTORRENT_USERNAME>` |
	| Password | `<YOUR_QBITTORRENT_PASSWORD>` | `<YOUR_QBITTORRENT_PASSWORD>` | `<YOUR_QBITTORRENT_PASSWORD>` |
	| Category | `movies` | `tv` | `anime` |
	| Tags | | | `anime` |
	| Remove Completed | ✅ | ✅ | ✅ |

	* [ ] Clicking on "Test" should return "✅"

	Save

8. In **[Sonarr](https://sonarr.tv/)**, go to `Settings → Tags → Auto Tagging → +`:

	| Field | Value |
	| --- | --- |
	| Name | `Anime` |
	| Remove Tags Automatically | ✅ |
	| Tags | `anime` |

	Then `Conditions → + → Root Folder`:

	| Field | Value |
	| --- | --- |
	| Name | `Anime root folder` |
	| Root Folder | `/data/media/anime/` |
	| Negate | ❌ |
	| Required | ✅ |

	Save

	* [ ] If you imported series in `/data/media/anime/` in step **6.**, they should show the `anime` tag after a few minutes (or after clicking on "Update All" in `Series`)

<br>

### V.&ensp;[Prowlarr](https://prowlarr.com/) configuration

1. In the **[Prowlarr](https://prowlarr.com/)** web interface, go to `Settings → Apps → Sync Profiles → Standard`:

	| Field | Value |
	| --- | --- |
	| Minimum Seeders | `10` |

	Save

2. Go to `Settings → Apps → Applications → + → Radarr`/`Sonarr`:

	| Field | **[Radarr](https://radarr.video/)** | **[Sonarr](https://sonarr.tv/)** |
	| --- | --- | --- |
	| Name | `Radarr` | `Sonarr` |
	| Sync Level | `Full Sync` | `Full Sync` |
	| Prowlarr Server | `http://gluetun:9696` | `http://gluetun:9696` |
	| Radarr / Sonarr Server | `http://172.20.0.20:7878` | `http://172.20.0.21:8989` |
	| API Key | `<YOUR_RADARR_API_KEY>` | `<YOUR_SONARR_API_KEY>` |

	* [ ] Clicking on "Test" should return "✅"

	Save

3. In **[Radarr](https://radarr.video/)** and **[Sonarr](https://sonarr.tv/)**, go to `Settings → Indexers`

	* [ ] You should see the indexers from **[Prowlarr](https://prowlarr.com/)**, with `Minimum Seeders` set to `10` when opening one of them (click on "Show Advanced")

<br>

### VI.&ensp;[Jellyfin](https://jellyfin.org/) configuration

1. In **[Jellyfin](https://jellyfin.org/)** (`tv.<YOUR_DOMAIN>`), go to `Dashboard → API Keys → +` and create one `Radarr` and one `Sonarr`

2. In **[Radarr](https://radarr.video/)** and **[Sonarr](https://sonarr.tv/)**, go to `Settings → Connect → + → Emby / Jellyfin`:

	| Field | **[Radarr](https://radarr.video/)** | **[Sonarr](https://sonarr.tv/)** |
	| --- | --- | --- |
	| Name | `Jellyfin` | `Jellyfin` |
	| Host | `jellyfin` | `jellyfin` |
	| Port | `8096` | `8096` |
	| API Key | `<YOUR_JELLYFIN_RADARR_API_KEY>` | `<YOUR_JELLYFIN_SONARR_API_KEY>` |
	| Send Notifications | ❌ | ❌ |
	| Update Library | ✅ | ✅ |

	* [ ] Clicking on "Test" should return "✅"

	Save

<br>

### VII.&ensp;Quality and profiles

1. On **[Radarr](https://radarr.video/)** and **[Sonarr](https://sonarr.tv/)**, you can configure the quality, language and other preferences for your media content, there is no standard configuration that fits everyone, but if you don't know where to start, you can check what I personally used: **[Radarr / Sonarr quality and profiles appendix](./appendices/radarr_sonarr_quality_and_profiles.md)**

<br>

### VIII.&ensp;Verifications

1. Use **[Radarr](https://radarr.video/)** or **[Sonarr](https://sonarr.tv/)** to download something

	* [ ] Everything should work and the content should appear in **[Jellyfin](https://jellyfin.org/)** after a short while (things can go wrong because of indexers or other issues, try again with different content if there is an issue, don't immediately blame your setup, also for new TV shows / anime you may need to wait for a **[Jellyfin](https://jellyfin.org/)** library scan or do it manually in the Dashboard)

2. Check the file management:

	```bash
	find /data/media/movies -type f -name '*.mkv' -newermt '-30 minutes' -exec stat -c '%h %i %n' {} \;
	find /data/torrents/movies -type f -name '*.mkv' -newermt '-30 minutes' -exec stat -c '%h %i %n' {} \;
	```

	* [ ] Both lines should start with `2` then another number, it should be the same number for both lines (replace `movies` with `tv` or `anime` if you used **[Sonarr](https://sonarr.tv/)**)

	<br>

	```bash
	ls -ln /data/media/movies/
	```

	* [ ] The files should have `1000` `1000` (or your user) as the owner and follow the `Title (year) [tmdbid-xxxxx]` format

	* [ ] In the **[qBittorrent](https://www.qbittorrent.org/)** web interface, the content should have the status `Seeding`

3. In **[Sonarr](https://sonarr.tv/)**, add an anime with `/data/media/anime/` as "Root Folder" and `Anime` as "Series Type", then download an episode

	* [ ] The series should get the `anime` tag

	* [ ] In **[qBittorrent](https://www.qbittorrent.org/)**, the download should appear under the `anime` category

	* [ ] When the download is complete, the episode should appear in the `Anime` library of **[Jellyfin](https://jellyfin.org/)** (not in `TV Shows`), and the hardlink check of step **2.** should pass with `anime`

<br>

### [Part 12: Bazarr](./12_bazarr.md)
