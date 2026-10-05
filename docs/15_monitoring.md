# Part 15: Monitoring

### I.&ensp;Notifications

1. In **[Discord](https://discord.com/)**, create a new server with a text channel in it

2. Set the notification settings to `All messages`

3. Go to the channel settings then `Integrations → Webhooks → New Webhook`, then click on `Copy Webhook URL` and save it

4. On your local computer, send a test notification:

	```bash
	curl -s -o /dev/null -w '%{http_code}\n' -H "Content-Type: application/json" -d '{"content":"Hello from mediabox"}' "<YOUR_DISCORD_WEBHOOK_URL>"
	```

	* [ ] The command should return `204` and you should see a message in the **[Discord](https://discord.com/)** channel

5. Add the webhook URL to the `.env`:

	```bash
	sudo tee -a /opt/mediaserver/.env > /dev/null <<'EOF'
	DISCORD_WEBHOOK_URL=<YOUR_DISCORD_WEBHOOK_URL>
	EOF
	```

6. On both the local computer and the VPS, install **[jq](https://jqlang.org/)**:

	```bash
	sudo apt update && sudo apt install -y jq
	```

<br>

### II.&ensp;Health notifications from [Radarr](https://radarr.video/), [Sonarr](https://sonarr.tv/) and [Prowlarr](https://prowlarr.com/)

1. In **[Radarr](https://radarr.video/)** and **[Sonarr](https://sonarr.tv/)**, go to `Settings → Connect → + → Discord`, in **[Prowlarr](https://prowlarr.com/)**, go to `Settings → Notifications → + → Discord`:

	| Field | **[Radarr](https://radarr.video/)** | **[Sonarr](https://sonarr.tv/)** | **[Prowlarr](https://prowlarr.com/)** |
	| --- | --- | --- | --- |
	| Name | `Discord` | `Discord` | `Discord` |
	| On Health Issue | ✅ | ✅ | ✅ |
	| On Health Restored | ✅ | ✅ | ✅ |
	| Include Health Warnings | ❌ | ❌ | ❌ |
	| On Manual Interaction Required | ✅ | ✅ | |
	| All the other "On ..." triggers | ❌ | ❌ | ❌ |
	| Webhook URL | `<YOUR_DISCORD_WEBHOOK_URL>` | `<YOUR_DISCORD_WEBHOOK_URL>` | `<YOUR_DISCORD_WEBHOOK_URL>` |
	| Username | `Radarr` | `Sonarr` | `Prowlarr` |
	| Avatar | **[\<This URL\>](https://raw.githubusercontent.com/angeluriot/Private_media_server/refs/heads/main/resources/images/radarr.png)** | **[\<This URL\>](https://raw.githubusercontent.com/angeluriot/Private_media_server/refs/heads/main/resources/images/sonarr.png)** | **[\<This URL\>](https://raw.githubusercontent.com/angeluriot/Private_media_server/refs/heads/main/resources/images/prowlarr.png)** |

	* [ ] Clicking on "Test" should return "✅" and a test message should appear in the **[Discord](https://discord.com/)** channel

	Save

<br>

### III.&ensp;Health checks on the local computer

1. Find the permanent name of the hard drive:

	```bash
	ls -l /dev/disk/by-id/ | grep -E "<YOUR_HDD_NAME>$"
	```

	Save the name starting with `ata-` (or `nvme-`, `usb-`, ..., but not `wwn-`), then:

	```bash
	sudo smartctl -H /dev/disk/by-id/<YOUR_HDD_ID>
	```

	* [ ] The command should return a `PASSED` result

2. Create the configuration of the health checks:

	```bash
	sudo tee /etc/mediaserver-check.env > /dev/null <<'EOF'
	DISCORD_WEBHOOK_URL=<YOUR_DISCORD_WEBHOOK_URL>
	DOMAIN=<YOUR_DOMAIN>
	HDD_ID=<YOUR_HDD_ID>
	EOF
	```

	```bash
	sudo chmod 600 /etc/mediaserver-check.env
	sudo chown root:root /etc/mediaserver-check.env
	```

3. Create the check script:

	```bash
	sudo nano /usr/local/bin/mediaserver-check
	```

	Write the content of **[scripts/local/mediaserver-check](../scripts/local/mediaserver-check)** in the file (it checks the disks, the failed services, the containers, the VPN port forwarding, the tunnel and the public access, and sends a message when something is wrong), then:

	```bash
	sudo chmod 755 /usr/local/bin/mediaserver-check
	```

4. Run it every 10 minutes:

	```bash
	sudo tee /etc/systemd/system/mediaserver-check.service > /dev/null <<'EOF'
	[Unit]
	Description=Media server health checks
	After=network-online.target docker.service
	Wants=network-online.target

	[Service]
	Type=oneshot
	EnvironmentFile=/etc/mediaserver-check.env
	ExecStart=/usr/local/bin/mediaserver-check
	EOF
	```

	```bash
	sudo tee /etc/systemd/system/mediaserver-check.timer > /dev/null <<'EOF'
	[Unit]
	Description=Run the media server health checks every 10 minutes

	[Timer]
	OnBootSec=10min
	OnUnitActiveSec=10min

	[Install]
	WantedBy=timers.target
	EOF
	```

	```bash
	sudo systemctl daemon-reload
	sudo systemctl enable --now mediaserver-check.timer
	```

5. Check the script:

	```bash
	sudo systemctl start mediaserver-check.service
	sudo journalctl -u mediaserver-check -n 5 --no-pager
	```

	* [ ] The logs should show `No problem detected`

6. Simulate a problem:

	```bash
	docker stop bazarr
	sudo systemctl start mediaserver-check.service
	sudo systemctl start mediaserver-check.service
	```

	* [ ] You should receive a `mediabox: problem detected` message saying `Container bazarr is exited`

	<br>

	```bash
	docker start bazarr
	sleep 90
	sudo systemctl start mediaserver-check.service
	sudo systemctl start mediaserver-check.service
	```

	* [ ] You should receive a `mediabox: back to normal` message

<br>

### IV.&ensp;Health checks on the VPS

1. On the VPS, create the configuration of the health checks:

	```bash
	sudo tee /etc/mediaserver-check.env > /dev/null <<'EOF'
	DISCORD_WEBHOOK_URL=<YOUR_DISCORD_WEBHOOK_URL>
	DOMAIN=<YOUR_DOMAIN>
	EOF
	```

	```bash
	sudo chmod 600 /etc/mediaserver-check.env
	sudo chown root:root /etc/mediaserver-check.env
	```

2. Create the check script:

	```bash
	sudo nano /usr/local/bin/mediaserver-check
	```

	Write the content of **[scripts/vps/mediaserver-check](../scripts/vps/mediaserver-check)** in the file (it checks the VPS services and the certificate, and also detects when the local computer is completely off or disconnected, which the local computer can't report by itself), then:

	```bash
	sudo chmod 755 /usr/local/bin/mediaserver-check
	```

3. Run it every 10 minutes:

	```bash
	sudo tee /etc/systemd/system/mediaserver-check.service > /dev/null <<'EOF'
	[Unit]
	Description=Media server health checks
	After=network-online.target
	Wants=network-online.target

	[Service]
	Type=oneshot
	EnvironmentFile=/etc/mediaserver-check.env
	ExecStart=/usr/local/bin/mediaserver-check
	EOF
	```

	```bash
	sudo tee /etc/systemd/system/mediaserver-check.timer > /dev/null <<'EOF'
	[Unit]
	Description=Run the media server health checks every 10 minutes

	[Timer]
	OnBootSec=10min
	OnUnitActiveSec=10min

	[Install]
	WantedBy=timers.target
	EOF
	```

	```bash
	sudo systemctl daemon-reload
	sudo systemctl enable --now mediaserver-check.timer
	```

4. Check the script:

	```bash
	sudo systemctl start mediaserver-check.service
	sudo journalctl -u mediaserver-check -n 5 --no-pager
	```

	* [ ] The logs should show `No problem detected`

5. Simulate a problem, on the local computer:

	```bash
	cd /opt/mediaserver
	docker compose stop seerr
	```

	Then on the VPS:

	```bash
	sudo systemctl start mediaserver-check.service
	sudo systemctl start mediaserver-check.service
	```

	* [ ] You should receive an `edge-vps: problem detected` message saying `Seerr is unreachable`

	On the local computer:

	```bash
	cd /opt/mediaserver
	docker compose start seerr
	```

	Wait a minute, then on the VPS:

	```bash
	sudo systemctl start mediaserver-check.service
	sudo systemctl start mediaserver-check.service
	```

	* [ ] You should receive an `edge-vps: back to normal` message (you may also receive messages from `mediabox` if its own timer ran in the meantime)

<br>

### V.&ensp;Final verification

1. Reboot the local computer and the VPS (`sudo reboot`), then on both:

	```bash
	systemctl list-timers mediaserver-check.timer
	```

	* [ ] A planned next run should be listed

	<br>

	```bash
	sudo systemctl start mediaserver-check.service
	sudo journalctl -u mediaserver-check -n 5 --no-pager
	```

	* [ ] The logs should show `No problem detected`

<br>

### [Part 16: Updates](./16_updates.md)
