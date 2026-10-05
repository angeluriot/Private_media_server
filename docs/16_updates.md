# Part 16: Updates

### I.&ensp;Container update notifications

1. On your local computer, create the **[What's Up Docker (WUD)](https://getwud.app/)** config directory:

	```bash
	sudo mkdir -p /opt/mediaserver/config/wud
	sudo chown -R 1000:1000 /opt/mediaserver/config/wud
	```

2. Choose a password for the **[WUD](https://getwud.app/)** web interface (without any `$` in it, Docker Compose would silently cut it) and add it to the `.env`:

	```bash
	sudo tee -a /opt/mediaserver/.env > /dev/null <<'EOF'
	WUD_ADMIN_PASSWORD=<A_PASSWORD_FOR_WUD>
	EOF
	```

3. Open `/opt/mediaserver/docker-compose.yml`, add the `labels` section to every existing service and add the **[WUD](https://getwud.app/)** service:

	```yaml
	services:
	  jellyfin:
	    ...
	    labels:
	      - wud.tag.include=^\d+\.\d+$$
	      - wud.link.template=https://github.com/jellyfin/jellyfin/releases
	  gluetun:
	    ...
	    labels:
	      - wud.tag.include=^v\d+\.\d+$$
	      - wud.link.template=https://github.com/qdm12/gluetun/releases
	  qbittorrent:
	    ...
	    labels:
	      - wud.tag.include=^\d\.\d+\.\d+$$
	      - wud.link.template=https://www.qbittorrent.org/news
	  prowlarr:
	    ...
	    labels:
	      - wud.tag.include=^version-\d+\.\d+\.\d+\.\d+$$
	      - wud.tag.transform=^version-(\d+\.\d+\.\d+)\.(\d+)$$ => $$1-$$2
	      - wud.link.template=https://github.com/Prowlarr/Prowlarr/releases
	  byparr:
	    ...
	    labels:
	      - wud.tag.include=^\d+\.\d+$$
	      - wud.link.template=https://github.com/ThePhaseless/Byparr/releases
	  radarr:
	    ...
	    labels:
	      - wud.tag.include=^version-\d+\.\d+\.\d+\.\d+$$
	      - wud.tag.transform=^version-(\d+\.\d+\.\d+)\.(\d+)$$ => $$1-$$2
	      - wud.link.template=https://github.com/Radarr/Radarr/releases
	  sonarr:
	    ...
	    labels:
	      - wud.tag.include=^version-\d+\.\d+\.\d+\.\d+$$
	      - wud.tag.transform=^version-(\d+\.\d+\.\d+)\.(\d+)$$ => $$1-$$2
	      - wud.link.template=https://github.com/Sonarr/Sonarr/releases
	  bazarr:
	    ...
	    labels:
	      - wud.tag.include=^version-v\d+\.\d+\.\d+$$
	      - wud.link.template=https://github.com/morpheus65535/bazarr/releases
	  seerr:
	    ...
	    labels:
	      - wud.tag.include=^v\d+\.\d+$$
	      - wud.link.template=https://github.com/seerr-team/seerr/releases
	  wud:
	    image: getwud/wud:9.2
	    container_name: wud
	    restart: unless-stopped
	    networks:
	      - media
	    ports:
	      - "3000:3000/tcp"
	    environment:
	      - TZ=${TZ}
	      - WUD_AUTH_ADMIN_USER=admin
	      - WUD_AUTH_ADMIN_PASSWORD=${WUD_ADMIN_PASSWORD}
	      - WUD_WATCHER_LOCAL_CRON=0 */6 * * *
	      - WUD_WATCHER_LOCAL_WATCHDIGESTDEFAULT=false
	      - WUD_TRIGGER_DISCORD_MEDIASERVER_URL=${DISCORD_WEBHOOK_URL}
	      - WUD_TRIGGER_DISCORD_MEDIASERVER_ONDIGEST=false
	      - WUD_TRIGGER_DISCORD_MEDIASERVER_CARDLABEL=Update
	      - "WUD_TRIGGER_DISCORD_MEDIASERVER_SIMPLETITLE=🆕 $${container.name}: new version $${container.updateKind.remoteValue}"
	      - |-
	        WUD_TRIGGER_DISCORD_MEDIASERVER_SIMPLEBODY=Release notes: $${container.result.link}
	        Command: \`sudo mediaserver-update $${container.name} $${container.updateKind.remoteValue}\`
	    volumes:
	      - /opt/mediaserver/config/wud:/store
	      - /var/run/docker.sock:/var/run/docker.sock:ro
	    labels:
	      - wud.tag.include=^\d+\.\d+$$
	      - wud.link.template=https://github.com/getwud/wud/releases
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

4. Check the Docker Compose configuration:

	```bash
	cd /opt/mediaserver
	docker compose config --quiet && echo "OK"
	```

	* [ ] The command should return `OK`

5. Apply the changes:

	```bash
	cd /opt/mediaserver
	docker compose up -d $(docker compose config --services | grep -vx wud)
	docker compose up -d wud
	until docker compose logs wud | grep -q "Cron finished"; do sleep 10; done
	docker compose logs wud | grep "Cron finished"
	```

	* [ ] The output should show `Cron finished (10 containers watched, 0 errors, ...)` (you may see new messages on **[Discord](https://discord.com/)**)

6. Go to **[localhost:3000](http://localhost:3000)** (or `http://<YOUR_LAN_IP>:3000` if you are on another computer in the LAN) and log in with `admin` and your **[WUD](https://getwud.app/)** password

	* [ ] The 10 containers should be listed, the ones with an update available should show the new tag

	* [ ] For each container with an update available, a message should have appeared in the **[Discord](https://discord.com/)** channel

<br>

### II.&ensp;Update script

1. On your local computer, create the update script:

	```bash
	sudo nano /usr/local/bin/mediaserver-update
	```

	Write the content of **[scripts/local/mediaserver-update](../scripts/local/mediaserver-update)** in the file (it does all the steps of a container update for you: backup, new tag, restart, health check, cleanup, and can go back to the previous version, the 3 most recent backups of each service are kept in `/data/backups`, you can change this number with `BACKUPS_TO_KEEP` at the top of the script), then:

	```bash
	sudo chmod 755 /usr/local/bin/mediaserver-update
	sudo mkdir -p /data/backups
	```

2. Check the script:

	```bash
	sudo mediaserver-update
	```

	* [ ] The command should print the usage

3. Test an update with **[Seerr](https://seerr.dev/)**, by switching from `v3.4` to `v3.4.1`:

	```bash
	sudo mediaserver-update seerr v3.4.1
	```

	* [ ] The last line should be `✅ seerr updated from ghcr.io/seerr-team/seerr:v3.4 to ghcr.io/seerr-team/seerr:v3.4.1`

	<br>

	```bash
	grep "image: ghcr.io/seerr-team" /opt/mediaserver/docker-compose.yml
	ls /data/backups/seerr-*/
	```

	* [ ] The first command should show the `v3.4.1` tag and the second should list `config.tar.gz` and `image`

4. Test the rollback:

	```bash
	sudo mediaserver-update --rollback seerr
	```

	* [ ] The last line should be `✅ seerr restored to ghcr.io/seerr-team/seerr:v3.4`

	<br>

	```bash
	grep "image: ghcr.io/seerr-team" /opt/mediaserver/docker-compose.yml
	```

	* [ ] The command should show the `v3.4` tag again

	* [ ] `https://request.<YOUR_DOMAIN>` should still work

<br>

### III.&ensp;Automatic container patches

1. On your local computer, apply the patches every week (replace `05:30` with a time you are most likely not watching anything, and at least 30 minutes away from the automatic reboot time):

	```bash
	sudo tee /etc/systemd/system/mediaserver-patch.service > /dev/null <<'EOF'
	[Unit]
	Description=Apply the updates republished on the current container tags
	After=network-online.target docker.service
	Wants=network-online.target

	[Service]
	Type=oneshot
	ExecStart=/usr/local/bin/mediaserver-update --patch
	EOF
	```

	```bash
	sudo tee /etc/systemd/system/mediaserver-patch.timer > /dev/null <<'EOF'
	[Unit]
	Description=Apply the container patches every week

	[Timer]
	OnCalendar=Mon *-*-* 05:30:00
	Persistent=true

	[Install]
	WantedBy=timers.target
	EOF
	```

	```bash
	sudo systemctl daemon-reload
	sudo systemctl enable --now mediaserver-patch.timer
	```

2. Run it once:

	```bash
	sudo systemctl start mediaserver-patch.service
	sudo journalctl -u mediaserver-patch -n 20 --no-pager
	```

	* [ ] The logs should end with `Everything is up to date` or `Updating: ...`, without any error

	<br>

	```bash
	cd /opt/mediaserver
	docker compose ps
	```

	* [ ] All containers should be `Up` (and `(healthy)` when they have a health check)

	<br>

	```bash
	systemctl list-timers mediaserver-patch.timer
	```

	* [ ] A planned next run should be listed

<br>

### IV.&ensp;Automatic updates of third-party packages on the local computer

1. **[unattended-upgrades](https://wiki.debian.org/UnattendedUpgrades)** only updates **[Debian](https://www.debian.org/)** packages for now, check the origins of Docker and **[CrowdSec](https://www.crowdsec.net/)**:

	```bash
	sudo apt-cache policy | grep -E "o=(Docker|packagecloud.io/crowdsec/crowdsec)," | sort -u
	```

	* [ ] The command should return a line containing `o=Docker` and `l=Docker CE`, and another one containing `o=packagecloud.io/crowdsec/crowdsec`

2. Open `/etc/apt/apt.conf.d/50unattended-upgrades` and add these 2 lines:

	```
	...
	Unattended-Upgrade::Origins-Pattern {
		...
		"origin=Docker,label=Docker CE";
		"origin=packagecloud.io/crowdsec/crowdsec";
	};
	...
	```

	Docker updates restart all the containers, it happens during the daily `apt-daily-upgrade` run (around 6 AM by default), so a stream can be cut for a few seconds at that time

3. Check the changes:

	```bash
	sudo unattended-upgrades --dry-run --debug 2>&1 | grep -i "allowed origins"
	```

	* [ ] The line should contain `origin=Docker,label=Docker CE` and `origin=packagecloud.io/crowdsec/crowdsec`

4. Check that the **[CrowdSec](https://www.crowdsec.net/)** collections, parsers and scenarios are updated automatically:

	```bash
	systemctl is-enabled crowdsec-hubupdate.timer
	systemctl list-timers crowdsec-hubupdate.timer
	```

	* [ ] The first command should return `enabled` and the second a planned next run

<br>

### V.&ensp;Automatic updates of third-party packages on the VPS

1. On the VPS, check the origin of **[CrowdSec](https://www.crowdsec.net/)**:

	```bash
	sudo apt-cache policy | grep -E "o=packagecloud.io/crowdsec/crowdsec," | sort -u
	```

	* [ ] The command should return a line containing `o=packagecloud.io/crowdsec/crowdsec`

2. Open `/etc/apt/apt.conf.d/50unattended-upgrades` and add this line:

	```
	...
	Unattended-Upgrade::Origins-Pattern {
		...
		"origin=packagecloud.io/crowdsec/crowdsec";
	};
	...
	```

3. Check the changes:

	```bash
	sudo unattended-upgrades --dry-run --debug 2>&1 | grep -i "allowed origins"
	```

	* [ ] The line should contain `origin=packagecloud.io/crowdsec/crowdsec`

4. Check that the **[CrowdSec](https://www.crowdsec.net/)** collections, parsers and scenarios are updated automatically:

	```bash
	systemctl is-enabled crowdsec-hubupdate.timer
	systemctl list-timers crowdsec-hubupdate.timer
	```

	* [ ] The first command should return `enabled` and the second a planned next run

5. **[Caddy](https://caddyserver.com/)** is blocked in `apt` since **[Part 5](./05_caddy_and_wildcard_tls.md)** to keep the **[Cloudflare](https://www.cloudflare.com/)** plugin, check that it's still the case:

	```bash
	apt-mark showhold
	```

	* [ ] The command should return `caddy`

6. Create a script that updates **[Caddy](https://caddyserver.com/)** with its plugins instead:

	```bash
	sudo nano /usr/local/bin/caddy-upgrade
	```

	Write the content of **[scripts/vps/caddy-upgrade](../scripts/vps/caddy-upgrade)** in the file, then:

	```bash
	sudo chmod 755 /usr/local/bin/caddy-upgrade
	```

7. Run it every week:

	```bash
	sudo tee /etc/systemd/system/caddy-upgrade.service > /dev/null <<'EOF'
	[Unit]
	Description=Upgrade Caddy with its plugins
	After=network-online.target
	Wants=network-online.target

	[Service]
	Type=oneshot
	ExecStart=/usr/local/bin/caddy-upgrade
	EOF
	```

	Replace `05:30` with a time you are most likely not watching anything:

	```bash
	sudo tee /etc/systemd/system/caddy-upgrade.timer > /dev/null <<'EOF'
	[Unit]
	Description=Upgrade Caddy every week

	[Timer]
	OnCalendar=Sun *-*-* 05:30:00
	Persistent=true

	[Install]
	WantedBy=timers.target
	EOF
	```

	```bash
	sudo systemctl daemon-reload
	sudo systemctl enable --now caddy-upgrade.timer
	```

8. Test the update:

	```bash
	sudo systemctl start caddy-upgrade.service
	systemctl show -p Result --value caddy-upgrade.service
	```

	* [ ] The last command should return `success`

	<br>

	```bash
	caddy list-modules | grep cloudflare
	systemctl is-active caddy
	```

	* [ ] The commands should return `dns.providers.cloudflare` and `active`

	<br>

	```bash
	systemctl list-timers caddy-upgrade.timer
	```

	* [ ] A planned next run should be listed

<br>

### VI.&ensp;Final verification

1. Reboot the local computer and the VPS (`sudo reboot`), then on the local computer:

	```bash
	systemctl list-timers mediaserver-patch.timer apt-daily-upgrade.timer
	```

	* [ ] Both timers should have a planned next run

	<br>

	```bash
	docker ps --filter name=wud --format '{{.Status}}'
	```

	* [ ] The command should return an `Up (healthy)` state

	On the VPS:

	```bash
	systemctl list-timers caddy-upgrade.timer apt-daily-upgrade.timer
	```

	* [ ] Both timers should have a planned next run

	<br>

	```bash
	apt-mark showhold
	```

	* [ ] The command should return `caddy`

2. On both, check that the health checks still don't see any problem:

	```bash
	sudo systemctl start mediaserver-check.service
	sudo journalctl -u mediaserver-check -n 5 --no-pager
	```

	* [ ] The logs should show `No problem detected`

<br>

### [Part 17: Maintenance](./17_maintenance.md)
