# Part 6: Entry point hardening

### I.&ensp;Configure [Caddy](https://caddyserver.com/)

1. On the VPS, open `/etc/caddy/Caddyfile` and add the following content:

	```caddyfile
	{ ... }

	*.<YOUR_DOMAIN> {
		tls { ... }
		log { ... }

		header {
			Strict-Transport-Security "max-age=31536000"
			X-Robots-Tag "noindex, nofollow, noarchive, nosnippet, noimageindex"
			X-Content-Type-Options "nosniff"
			X-Frame-Options "SAMEORIGIN"
			Referrer-Policy "no-referrer"
			-Server
		}

		handle /robots.txt {
			header Content-Type "text/plain"
			respond <<ROBOTS
				User-agent: *
				Disallow: /
				ROBOTS 200
		}

		@tv host tv.<YOUR_DOMAIN>
		handle @tv { ... }

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

4. On your local computer, check the changes:

	```bash
	curl -sI https://tv.<YOUR_DOMAIN> | grep -iE "x-robots-tag|strict-transport|x-content-type|x-frame|referrer-policy"
	```

	* [ ] The command should return the headers you set in the **[Caddy](https://caddyserver.com/)** configuration

	<br>

	```bash
	curl -sI https://tv.<YOUR_DOMAIN> | grep -i "^server:"
	```

	* [ ] The command should return nothing

	<br>

	```bash
	curl -s https://tv.<YOUR_DOMAIN>/robots.txt
	```

	* [ ] The command should return:

	<br>

	```
	User-agent: *
	Disallow: /
	```

	```bash
	curl -s https://tv.<YOUR_DOMAIN>
	```

	* [ ] The command should return `tv ok`

<br>

### II.&ensp;Set up [CrowdSec](https://www.crowdsec.net/)

1. Install **[CrowdSec](https://www.crowdsec.net/)** on the VPS:

	```bash
	curl -s https://install.crowdsec.net | sudo sh
	sudo apt update && sudo apt install -y crowdsec
	```

2. Check the installation:

	```bash
	sudo systemctl status crowdsec --no-pager
	```

	* [ ] The command should show `active (running)` and no errors

	<br>

	```bash
	sudo cscli version
	```

	* [ ] The command should show the version

	<br>

	```bash
	sudo cscli hub list
	```

	* [ ] The command should show the list of collections and parsers

3. Ensure that **[CrowdSec](https://www.crowdsec.net/)** can read the **[Caddy](https://caddyserver.com/)** logs:

	```bash
	sudo grep -rn "caddy" /etc/crowdsec/acquis.yaml /etc/crowdsec/acquis.d/ 2>/dev/null
	```

	* [ ] The command should find matching lines in `/etc/crowdsec/acquis.d/setup.caddy.yaml`

4. Ensure that these collections and parsers are installed:

	```bash
	sudo cscli collections install crowdsecurity/caddy
	sudo cscli collections install crowdsecurity/http-cve
	sudo cscli parsers install crowdsecurity/whitelists
	```

5. Restart **[CrowdSec](https://www.crowdsec.net/)**:

	```bash
	sudo systemctl restart crowdsec
	sudo systemctl status crowdsec --no-pager
	```

	* [ ] The command should show `active (running)` and no errors

<br>

### III.&ensp;Verify [CrowdSec](https://www.crowdsec.net/) is working

1. On your local computer:

	```bash
	curl -s https://tv.<YOUR_DOMAIN> > /dev/null
	curl -s https://tv.<YOUR_DOMAIN>/robots.txt > /dev/null
	```

2. Wait a few seconds, then on the VPS:

	```bash
	sudo cscli metrics
	```

	* [ ] In the `Acquisition Metrics` section, the line with `caddy` should have a non-zero number of lines read

<br>

### IV.&ensp;Whitelist your IP address

1. On your local computer, get and save your IPv4 address and your IPv6 prefix:

	```bash
	curl -s -4 https://ifconfig.me
	curl -s -6 https://ifconfig.me | sed -E 's/(([0-9a-f]{1,4}:){4}).*/\1:\/56/'
	```

2. On the VPS, create a whitelist file:

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
	    - "<YOUR_IPV6_PREFIX>"
	EOF
	```

	```bash
	sudo systemctl reload crowdsec
	```

3. Check that your IP address is whitelisted:

	```bash
	sudo cscli parsers list | grep my-whitelist
	```

	* [ ] The command should list `custom/my-whitelist` with `enabled` status

	<br>

	```bash
	sudo journalctl -u crowdsec -n 30 --no-pager | grep -i error
	```

	* [ ] The command should return nothing

<br>

### V.&ensp;Set up the bouncer

1. Install the bouncer on the VPS:

	```bash
	sudo apt update && sudo apt install -y crowdsec-firewall-bouncer-nftables
	```

2. Open `/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml` and change `update_frequency` from `10s` to `5s`:

	```yaml
	...
	update_frequency: 5s
	...
	```

3. Restart the bouncer:

	```bash
	sudo systemctl restart crowdsec-firewall-bouncer
	```

4. Check that the bouncer is running:

	```bash
	sudo systemctl status crowdsec-firewall-bouncer --no-pager
	```

	* [ ] The command should show `active (running)` and no errors

	<br>

	```bash
	sudo cscli bouncers list
	```

	* [ ] The command should list a `Valid` bouncer with a recent `Last API pull`

	<br>

	```bash
	sudo nft -a list tables
	```

	* [ ] Two `crowdsec` tables should be listed

5. Ban an IP address to test the bouncer:

	```bash
	sudo cscli decisions add --ip 192.0.2.1 --duration 2m --reason "test"
	sudo cscli decisions list
	```

	* [ ] The last command should list `192.0.2.1` as a `ban`

	<br>

	```bash
	sleep 10 && sudo nft list table ip crowdsec | grep 192.0.2.1
	```

	* [ ] The command should show `192.0.2.1`

	<br>

	```bash
	sudo cscli decisions delete --ip 192.0.2.1
	```

<br>

### VI.&ensp;Verifications

1. Open `/etc/caddy/Caddyfile` and add the following content:

	```caddyfile
	{ ... }

	*.<YOUR_DOMAIN> {
		tls { ... }
		log { ... }
		header { ... }
		handle /robots.txt { ... }

		@tv host tv.<YOUR_DOMAIN>
		handle @tv {
			handle / {
				respond "tv ok" 200
			}
			handle {
				respond "not found" 404
			}
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

4. Disconnect your computer from your router and use a 5G hotspot instead (you can also use another device), then:

	```bash
	for i in $(seq 1 50); do
	  curl -s -o /dev/null "https://tv.<YOUR_DOMAIN>/scan-test-$i"
	done
	```

5. On the VPS, check the alerts:

	```bash
	sudo cscli alerts list
	```

	* [ ] A line with your mobile operator and your phone's IP address should be listed with a reason like `crowdsecurity/http-probing`

6. Check the decisions:

	```bash
	sudo cscli decisions list
	```

	* [ ] A line with your mobile operator and your phone's IP address should be listed with an action `ban`

7. On your device connected to the 5G network:

	```bash
	curl -s --max-time 5 https://tv.<YOUR_DOMAIN> || echo "BANNED"
	```

	* [ ] The command should return `BANNED`

8. On the VPS, clean up all decisions:

	```bash
	sudo cscli decisions delete --all
	sudo cscli decisions list
	```

	* [ ] The last command should return `No active decisions`

9. Open `/etc/caddy/Caddyfile` and revert the changes:

	```caddyfile
	{ ... }

	*.<YOUR_DOMAIN> {
		tls { ... }
		log { ... }
		header { ... }
		handle /robots.txt { ... }

		@tv host tv.<YOUR_DOMAIN>
		handle @tv {
			respond "tv ok" 200
		}

		@request host request.<YOUR_DOMAIN>
		handle @request { ... }

		handle { ... }
	}
	```

10. Check the syntax:

	```bash
	sudo caddy fmt --overwrite /etc/caddy/Caddyfile
	sudo caddy adapt --config /etc/caddy/Caddyfile > /dev/null && echo "Syntax OK"
	```

	* [ ] The last command should return `Syntax OK`

11. Reload **[Caddy](https://caddyserver.com/)**:

	```bash
	sudo systemctl reload caddy
	sudo systemctl status caddy --no-pager
	```

	* [ ] The last command should show `active (running)` and no errors

12. On your device connected to the 5G network:

	```bash
	curl -s --max-time 5 https://tv.<YOUR_DOMAIN> || echo "BANNED"
	```

	* [ ] The command should return `tv ok` (you may need to wait a few minutes)

13. The 5G hotspot is no longer needed, reboot the VPS:

	```bash
	sudo reboot
	```

14. Then on the VPS, check the status of the services:

	```bash
	systemctl is-active caddy crowdsec crowdsec-firewall-bouncer wg-quick@wg0
	```

	* [ ] All four services should be `active`

	Wait a few seconds, then:

	```bash
	sudo wg show
	```

	* [ ] The `latest handshake` should be recent

	<br>

	```bash
	sudo ls -l /proc/$(pgrep -x crowdsec)/fd | grep caddy
	```

	* [ ] The file should be `/var/log/caddy/access.log`

15. On your local computer:

	```bash
	curl -sI https://tv.<YOUR_DOMAIN> | grep -i x-robots-tag
	curl -s https://tv.<YOUR_DOMAIN>
	```

	* [ ] The commands should return the headers and `tv ok` respectively

16. Go to **[securityheaders.com](https://securityheaders.com/)** and enter `tv.<YOUR_DOMAIN>`, you should get something like `B` with some headers but not all

<br>

### [Part 7: Jellyfin](./07_jellyfin.md)
