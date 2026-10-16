# Part 7: [Jellyfin](https://jellyfin.org/)

### I.&ensp;Add test data

1. On your local computer, download **[Big Buck Bunny (2008)](https://en.wikipedia.org/wiki/Big_Buck_Bunny)**:

	```bash
	sudo apt update && sudo apt install -y wget unzip
	mkdir -p "/data/media/movies/Big Buck Bunny (2008)"
	cd "/data/media/movies/Big Buck Bunny (2008)"
	wget https://download.blender.org/peach/bigbuckbunny_movies/big_buck_bunny_1080p_h264.mov.zip
	unzip big_buck_bunny_1080p_h264.mov.zip
	rm big_buck_bunny_1080p_h264.mov.zip
	docker run --rm --user 1000:1000 -v "/data/media:/data/media" --entrypoint /usr/lib/jellyfin-ffmpeg/ffmpeg jellyfin/jellyfin:10.11 -i "/data/media/movies/Big Buck Bunny (2008)/big_buck_bunny_1080p_h264.mov" -c copy -movflags +faststart "/data/media/movies/Big Buck Bunny (2008)/Big Buck Bunny (2008) Bluray-1080p.mp4"
	rm big_buck_bunny_1080p_h264.mov
	ls -l "/data/media/movies/Big Buck Bunny (2008)"
	```

	* [ ] The last command should show a `Big Buck Bunny (2008) Bluray-1080p.mp4` file belonging to `1000` `1000` (or your user)

<br>

### II.&ensp;Create the configuration files

1. Create the `.env`:

	```bash
	sudo tee /opt/mediaserver/.env > /dev/null <<'EOF'
	PUID=1000
	PGID=1000
	TZ=<YOUR_TIMEZONE>
	RENDER_GID=<YOUR_RENDER_GID>
	EOF
	```

	```bash
	sudo chmod 600 /opt/mediaserver/.env
	sudo chown 1000:1000 /opt/mediaserver/.env
	```

2. Create `/opt/mediaserver/docker-compose.yml` and put the following content:

	```yaml
	services:
	  jellyfin:
	    image: jellyfin/jellyfin:10.11
	    container_name: jellyfin
	    hostname: <A_NAME_FOR_YOUR_MEDIA_SERVER>
	    user: "${PUID}:${PGID}"
	    group_add:
	      - "${RENDER_GID}"
	    restart: unless-stopped
	    networks:
	      - media
	    environment:
	      - TZ=${TZ}
	      - JELLYFIN_PublishedServerUrl=https://tv.<YOUR_DOMAIN>
	    ports:
	      - "8096:8096/tcp"
	    devices:
	      - /dev/dri/renderD128:/dev/dri/renderD128
	    volumes:
	      - /opt/mediaserver/config/jellyfin:/config
	      - /opt/mediaserver/cache/jellyfin:/cache
	      - type: bind
	        source: /data/media
	        target: /data/media
	        read_only: true
	    deploy:
	      resources:
	        limits:
	          memory: 3G
	    logging:
	      driver: json-file
	      options:
	        max-size: "10m"
	        max-file: "3"
	networks:
	  media:
	    name: media
	    ipam:
	      config:
	        - subnet: 172.20.0.0/24
	          gateway: 172.20.0.1
	```

3. Check the Docker Compose configuration:

	```bash
	cd /opt/mediaserver
	docker compose config --quiet && echo "OK"
	```

	* [ ] The command should return `OK`

<br>

### III.&ensp;First startup

1. Start the **[Jellyfin](https://jellyfin.org/)** container:

	```bash
	cd /opt/mediaserver
	docker compose up -d
	docker compose logs -f jellyfin
	```

	* [ ] The logs should stop with something like `Main: Startup complete`

2. Check the health of the **[Jellyfin](https://jellyfin.org/)** container:

	```bash
	curl -s http://localhost:8096/health
	```

	* [ ] The command should return `Healthy`

	<br>

	```bash
	cd /opt/mediaserver
	docker compose ps
	```

	* [ ] `8096->8096/tcp` should be visible in the `PORTS` column

	<br>

	```bash
	docker inspect media -f '{{range .IPAM.Config}}{{.Gateway}}{{end}}'
	```

	* [ ] The command should return `172.20.0.1`

3. Check that the **[Intel](https://www.intel.com/)** **[iGPU](https://en.wikipedia.org/wiki/List_of_Intel_graphics_processing_units)** is usable from the **[Jellyfin](https://jellyfin.org/)** container:

	```bash
	docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
	```

	* [ ] The command should show `Intel iHD driver` without any error, followed by a list of `VAProfile...` lines, some of them ending with `VAEntrypointEncSliceLP`

	<br>

	```bash
	docker exec jellyfin /usr/lib/jellyfin-ffmpeg/ffmpeg -v verbose -init_hw_device vaapi=va:/dev/dri/renderD128 -init_hw_device opencl@va 2>&1 | grep -i opencl
	```

	* [ ] The command should show a line with `Intel(R) OpenCL Graphics`

4. Open your web browser and go to **[localhost:8096](http://localhost:8096)** to access the **[Jellyfin](https://jellyfin.org/)** web interface (or `http://<YOUR_LAN_IP>:8096` if you are on another computer in the LAN)

5. The server name should be set to the name you chose as your `hostname` in the Docker Compose and you can choose the language you prefer

6. Choose the administrator username and password for your **[Jellyfin](https://jellyfin.org/)** server

7. Add three libraries for your media content:

	| Content type | Display name | Folders |
	| --- | --- | --- |
	| `Movies` | `Movies` | `/data/media/movies` |
	| `Shows` | `TV Shows` | `/data/media/tv` |
	| `Shows` | `Anime` | `/data/media/anime` |

	For each of them, set these options:

	| Field | Value |
	| --- | --- |
	| Preferred download language | \<Your language\> |
	| Country/Region | \<Your country\> |
	| Enable real time monitoring | ✅ |
	| Metadata savers → Nfo | ❌ |
	| Save artwork into media folders | ❌ |
	| Enable trickplay image extraction | ✅ |
	| Extract trickplay images during the library scan | ✅ |
	| Save trickplay images next to media | ❌ |
	| Enable chapter image extraction | ❌ |

8. Choose your preferred metadata language and country

9. Check "Allow remote connections to this server"

10. In `Dashboard → Networking`:

	* Known proxies: `10.10.0.1, 172.20.0.1`
	* Published server URIs: `https://tv.<YOUR_DOMAIN>`

	Save

11. Restart the **[Jellyfin](https://jellyfin.org/)** container to apply the changes:

	```bash
	cd /opt/mediaserver
	docker compose restart jellyfin
	```

12. Check the status of the **[Jellyfin](https://jellyfin.org/)** container to ensure it is running correctly:

	```bash
	cd /opt/mediaserver
	docker compose ps
	```

	* [ ] The `jellyfin` container should have a `STATUS` of `Up`

<br>

### IV.&ensp;Connect [Jellyfin](https://jellyfin.org/) to the VPS

1. On the VPS, open `/etc/caddy/Caddyfile` and add the following content:

	```caddyfile
	{ ... }

	*.<YOUR_DOMAIN> {
		tls { ... }

		log {
			output file /var/log/caddy/access.log
			format filter {
				wrap json
				fields {
					request>uri query {
						replace api_key REDACTED
						replace ApiKey REDACTED
					}
				}
			}
		}

		header { ... }
		handle /robots.txt { ... }

		@tv host tv.<YOUR_DOMAIN>
		handle @tv {
			reverse_proxy 10.10.0.2:8096
		}

		@request host request.<YOUR_DOMAIN>
		handle @request { ... }

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

4. Using your phone, switch to 5G and go to `https://tv.<YOUR_DOMAIN>` to access your **[Jellyfin](https://jellyfin.org/)** server remotely, then enter the administrator username and a wrong password

5. On your local computer:

	```bash
	docker exec jellyfin sh -c 'grep -i "has been denied" /config/log/log_*.log | tail -5'
	```

	* [ ] You should see at the end `Authentication request for "<YOUR_ADMIN_USERNAME>" has been denied (IP: "<YOUR_MOBILE_IP>")` (you can find your mobile IP by searching "my ip" on Google)

6. On the VPS:

	```bash
	sudo grep -oE 'api_key=[^&"]*' /var/log/caddy/access.log | grep -vc 'api_key=REDACTED'
	```

	* [ ] The command should return `0`

7. On the VPS, install the **[Jellyfin](https://jellyfin.org/)** whitelist:

	```bash
	sudo cscli parsers install crowdsecurity/jellyfin-whitelist
	```

8. Restart **[CrowdSec](https://www.crowdsec.net/)**:

	```bash
	sudo systemctl restart crowdsec
	```

9. Check that **[CrowdSec](https://www.crowdsec.net/)** is still running:

	```bash
	sudo systemctl status crowdsec --no-pager
	```

	* [ ] The command should show `active (running)`

<br>

### V.&ensp;Brute-force protection

1. On the VPS, open `/etc/crowdsec/config.yaml` and add/edit the following content:

	```yaml
	...
	api:
	  ...
	  server:
	    ...
	    listen_uri: 0.0.0.0:8080
	    ...
	    trusted_ips:
	      - 127.0.0.1
	      - ::1
	      - 10.10.0.2
	...
	```

	Then:

	```bash
	sudo ufw allow from 10.10.0.2 to any port 8080 proto tcp comment 'CrowdSec LAPI'
	sudo systemctl restart crowdsec
	```

2. Verifications:

	```bash
	sudo ss -tulpn | grep 8080
	```

	* [ ] You should see `*:8080` or `0.0.0.0:8080`

	<br>

	```bash
	sudo cscli lapi status
	```

	* [ ] The command should return a successful status

	<br>

	```bash
	sudo ufw status numbered
	```

	* [ ] The command should show a line with `10.10.0.2`

3. On the local computer, install **[CrowdSec](https://www.crowdsec.net/)**:

	```bash
	curl -s https://install.crowdsec.net | sudo sh
	sudo apt update && sudo apt install -y crowdsec
	```

4. Test the LAPI connection:

	```bash
	curl -s -o /dev/null -w '%{http_code}\n' http://10.10.0.1:8080/health
	```

	* [ ] The command should return an HTTP status code like `200` and not `connection refused`

5. Register the local computer:

	```bash
	sudo cscli lapi register --machine mediabox -u http://10.10.0.1:8080
	sudo cat /etc/crowdsec/local_api_credentials.yaml
	```

	* [ ] The last command should show `http://10.10.0.1:8080` for the `url`

6. On the VPS, verify the registration:

	```bash
	sudo cscli machines validate mediabox
	sudo cscli machines list
	```

	* [ ] The last command should show the local computer (`mediabox`) with a validated status

7. On the local computer, open `/etc/crowdsec/config.yaml` and add the following content:

	```yaml
	...
	api:
	  ...
	  server:
	    enable: false
	  ...
	...
	```

	Then:

	```bash
	sudo systemctl restart crowdsec
	sudo ss -tulpn | grep 8080
	```

	* [ ] The last command should return nothing

	<br>

	```bash
	sudo cscli lapi status
	```

	* [ ] The command should return a successful status

8. Add standard **[CrowdSec](https://www.crowdsec.net/)** collections on the local computer:

	```bash
	sudo cscli collections install crowdsecurity/linux
	```

9. Add the **[Jellyfin](https://jellyfin.org/)** collection on the local computer:

	```bash
	sudo cscli collections install LePresidente/jellyfin
	```

	```bash
	sudo tee /etc/crowdsec/acquis.d/jellyfin.yaml > /dev/null <<'EOF'
	source: docker
	container_name:
	  - jellyfin
	labels:
	  type: jellyfin
	EOF
	```

10. Add a local **[CrowdSec](https://www.crowdsec.net/)** whitelist:

	```bash
	sudo tee /etc/crowdsec/parsers/s02-enrich/mywhitelist.yaml > /dev/null <<'EOF'
	name: custom/my-whitelist
	description: "Trusted IPs that must never be banned"
	whitelist:
	  reason: "my own addresses"
	  ip:
	    - "<YOUR_IPV4_ADDRESS>"
	  cidr:
	    - "10.10.0.0/24"
	    - "172.20.0.0/24"
	    - "<YOUR_IPV6_PREFIX>"
	EOF
	```

	```bash
	sudo systemctl restart crowdsec
	```

11. Verify the local whitelist:

	```bash
	sudo systemctl status crowdsec --no-pager
	```

	* [ ] The command should show `active (running)`

	<br>

	```bash
	sudo cscli parsers list | grep my-whitelist
	```

	* [ ] `custom/my-whitelist` should be listed and `enabled`

	<br>

	```bash
	sudo journalctl -u crowdsec -n 30 --no-pager | grep -i error
	```

	* [ ] The command should return nothing

12. Using your phone, switch to 5G and go to `https://tv.<YOUR_DOMAIN>` to access your **[Jellyfin](https://jellyfin.org/)** server remotely, then enter the administrator username and try wrong passwords repeatedly

	* [ ] After some tries, the website should freeze and stop working

13. On your local computer:

	```bash
	sudo cscli metrics
	```

	* [ ] In `Acquisition Metrics`, the lines read for `docker:jellyfin` should be non-zero

14. On the VPS:

	```bash
	sudo cscli alerts list
	```

	* [ ] A line with your phone IP should be listed

	<br>

	```bash
	sudo cscli decisions list
	```

	* [ ] A line with your phone IP should be listed with the action `ban`

	<br>

	```bash
	sudo cscli decisions delete --all
	```

	* [ ] After waiting a few minutes, you should have access to the website again with your phone IP

<br>

### VI.&ensp;Transcoding

1. On the local computer, list the codecs the **[iGPU](https://en.wikipedia.org/wiki/List_of_Intel_graphics_processing_units)** can decode:

	```bash
	docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128 2>/dev/null | grep VAEntrypointVLD
	```

2. In **[Jellyfin](https://jellyfin.org/)**, go to `Dashboard → Playback → Transcoding`:

	Select `Intel QuickSync (QSV)` for "Hardware acceleration", leave "QSV Device" empty, then for "Enable hardware decoding for", only check a codec if its profile appeared in the output of step **1.**:

	| Codec | Profile |
	| --- | --- |
	| H264 | `VAProfileH264High` |
	| HEVC | `VAProfileHEVCMain` |
	| MPEG2 | `VAProfileMPEG2Main` |
	| VC1 | `VAProfileVC1Advanced` |
	| VP8 | `VAProfileVP8Version0_3` |
	| VP9 | `VAProfileVP9Profile0` |
	| AV1 | `VAProfileAV1Profile0` |
	| HEVC 10bit | `VAProfileHEVCMain10` |
	| VP9 10bit | `VAProfileVP9Profile2` |
	| HEVC RExt 8/10bit | `VAProfileHEVCMain422_10` |
	| HEVC RExt 12bit | `VAProfileHEVCMain12` |

	Then:

	| Field | Value |
	| --- | --- |
	| Prefer OS native DXVA or VA-API hardware decoders | ✅ |
	| Enable hardware encoding | ✅ |
	| Enable Intel Low-Power H.264 hardware encoder | ✅ |
	| Enable Intel Low-Power HEVC hardware encoder | ✅ |
	| Allow encoding in HEVC format | ❌ |
	| Allow encoding in AV1 format | ❌ |
	| Enable VPP Tone mapping | ❌ |
	| Enable Tone mapping | ✅ |
	| Throttle Transcodes | ✅ |
	| Delete segments | ✅ |

	Save

3. In `Dashboard → Playback → Trickplay`:

	| Field | Value |
	| --- | --- |
	| Enable hardware decoding | ✅ |
	| Enable hardware accelerated MJPEG encoding | ✅ |
	| Only generate images from key frames | ✅ |

	Save

4. In **[Jellyfin](https://jellyfin.org/)**, play **[Big Buck Bunny (2008)](https://en.wikipedia.org/wiki/Big_Buck_Bunny)**, then in the player settings set `Quality` to `420 kbps`

	* [ ] In `Playback Info` you should see `Play method Transcoding`

5. Check the transcoding logs:

	```bash
	docker exec jellyfin sh -c 'f=$(ls -t /config/log/FFmpeg.Transcode-*.log | head -1); grep -oE "\-hwaccel(_output_format)? [a-z]+|hwmap=derive_device=[a-z]+|\(h264_qsv\)" "$f" | sort -u'
	```

	* [ ] The command should return `(h264_qsv)`, `-hwaccel_output_format vaapi`, `-hwaccel vaapi` and `hwmap=derive_device=qsv`

<br>

### VII.&ensp;Configuration

1. In `Dashboard → Plugins`, install these plugins:

	| Plugin | Purpose |
	| --- | --- |
	| `TheTVDB` | Better metadata for shows, matching the episode numbering used by **[Sonarr](https://sonarr.tv/)** |
	| `Chapter Segments Provider` | Adds a skip button for openings, endings, recaps and previews, based on chapter names |
	| `Playback Reporting` | Statistics on who watched what, when, and how |
	| `Fanart` | Additional artwork |

2. Restart **[Jellyfin](https://jellyfin.org/)**:

	```bash
	cd /opt/mediaserver
	docker compose restart jellyfin
	```

	* [ ] In `Dashboard → Plugins`, these four plugins should be listed as `Active`

3. In `Dashboard → Libraries → Libraries`, for both `TV Shows` and `Anime`, click on `⋮ → Manage library`:

	* In all `Metadata downloaders` sections, check `TheTVDB` and move it to the top
	* In all `Image fetchers` sections, check `TheTVDB` and move it to the top

	Save

4. In `Dashboard → Libraries → Libraries`, for all three libraries, click on `⋮ → Manage library`:

	* In all `Image fetchers` sections, check `Fanart` and move it to the bottom
	* In `Media segment providers`, check `Chapter Segments Provider`

	Save

5. Click on `Dashboard → Scheduled Tasks → Media Segment Scan → ▶`

6. Play any media, then go to `Dashboard → Playback Reporting`

	* [ ] The media you just started should be listed in the activity

7. In `Dashboard → Scheduled Tasks → Scan Media Library → +`:

	| Field | Value |
	| --- | --- |
	| Trigger type | `On an interval` |
	| Every | `15 minutes` |
	| Time limit (hours) | |

	Then delete the old "Every 12 hours"

<br>

### VIII.&ensp;Verification

1. Reboot the local computer, then the VPS (`sudo reboot`), and each time check that everything still works, then on the local computer:

	```bash
	docker ps --filter name=jellyfin --format '{{.Status}}'
	```

	* [ ] The command should return an `Up` state

	<br>

	```bash
	docker inspect media -f '{{range .IPAM.Config}}{{.Gateway}}{{end}}'
	```

	* [ ] The command should return `172.20.0.1`

	<br>

	```bash
	docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
	```

	* [ ] The command should show `Intel iHD driver` without any error

	<br>

	```bash
	systemctl is-active crowdsec
	```

	* [ ] The command should return `active`

	On the VPS:

	```bash
	sudo cscli machines list
	```

	* [ ] A `mediabox` line with a validated status and a recent heartbeat should be listed

<br>

### [Part 8: Gluetun](./08_gluetun.md)
