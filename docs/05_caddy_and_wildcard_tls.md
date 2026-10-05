# Part 5: [Caddy](https://caddyserver.com/) and wildcard TLS

### I.&ensp;Set up [Caddy](https://caddyserver.com/)

1. Install **[Caddy](https://caddyserver.com/)** on the VPS:

	```bash
	sudo apt update && sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
	curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
	curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
	sudo apt update && sudo apt install -y caddy
	```

2. Install the **[Cloudflare](https://www.cloudflare.com/)** plugin:

	```bash
	sudo caddy add-package github.com/caddy-dns/cloudflare
	sudo systemctl restart caddy
	caddy list-modules | grep cloudflare
	```

	* [ ] The last command should return `dns.providers.cloudflare`

3. Prevent `apt` from replacing **[Caddy](https://caddyserver.com/)** with a version without the plugin:

	```bash
	sudo apt-mark hold caddy
	apt-mark showhold
	```

	* [ ] The last command should return `caddy`

4. Add the **[Cloudflare](https://www.cloudflare.com/)** API token in the environment file:

	```bash
	sudo tee /etc/caddy/caddy.env > /dev/null <<'EOF'
	CLOUDFLARE_API_TOKEN=<YOUR_CLOUDFLARE_API_TOKEN>
	EOF
	```

	```bash
	sudo chmod 600 /etc/caddy/caddy.env
	sudo chown root:root /etc/caddy/caddy.env
	```

5. Add the token path in the service file:

	```bash
	sudo mkdir -p /etc/systemd/system/caddy.service.d
	```

	```bash
	sudo tee /etc/systemd/system/caddy.service.d/override.conf > /dev/null <<'EOF'
	[Service]
	EnvironmentFile=/etc/caddy/caddy.env
	ExecStart=
	ExecStart=/usr/bin/caddy run --config /etc/caddy/Caddyfile
	LogsDirectory=caddy
	LogsDirectoryMode=0750
	EOF
	```

	```bash
	sudo systemctl daemon-reload
	```

6. Check the service file:

	```bash
	sudo systemctl show caddy -p ExecStart | grep -c argv
	```

	* [ ] The command should return `1`

	<br>

	```bash
	sudo systemctl show caddy -p ExecStart | grep environ
	```

	* [ ] The command should return nothing

<br>

### II.&ensp;Add the new [UFW](https://wiki.ubuntu.com/UncomplicatedFirewall) rules

1. On the VPS, allow HTTP and HTTPS ports:

	```bash
	sudo ufw allow 80/tcp comment 'HTTP redirect'
	sudo ufw allow 443/tcp comment 'HTTPS'
	sudo ufw allow 443/udp comment 'HTTP/3'
	```

2. Check the new rules:

	```bash
	sudo ufw status numbered
	```

	* [ ] The command should show the new rules

<br>

### III.&ensp;Configure the Caddyfile

1. Open `/etc/caddy/Caddyfile` and replace everything with the following content:

	```caddyfile
	{
		email <YOUR_EMAIL_ADDRESS>
		acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
	}

	*.<YOUR_DOMAIN> {
		tls {
			dns cloudflare {env.CLOUDFLARE_API_TOKEN}
			resolvers 1.1.1.1 8.8.8.8
			propagation_timeout 5m
		}

		log {
			output file /var/log/caddy/access.log
			format json
		}

		@tv host tv.<YOUR_DOMAIN>
		handle @tv {
			respond "tv ok" 200
		}

		@request host request.<YOUR_DOMAIN>
		handle @request {
			respond "request ok" 200
		}

		handle {
			abort
		}
	}
	```

2. Check the syntax:

	```bash
	sudo caddy fmt --overwrite /etc/caddy/Caddyfile
	sudo caddy adapt --config /etc/caddy/Caddyfile > /dev/null && echo "Syntax OK"
	```

	* [ ] The last command should return `Syntax OK`

<br>

### IV.&ensp;Start [Caddy](https://caddyserver.com/)

1. Restart **[Caddy](https://caddyserver.com/)**:

	```bash
	sudo systemctl restart caddy
	sudo journalctl -u caddy -f
	```

	* [ ] The last command should show `certificate obtained successfully` and NOT `server is listening only on the HTTP port`

2. Verifications:

	```bash
	systemctl status caddy --no-pager
	```

	* [ ] The command should show `active (running)` and no errors

	<br>

	```bash
	ls -ld /var/log/caddy
	```

	* [ ] The command should show `caddy caddy` as owner and group

	<br>

	```bash
	sudo journalctl -u caddy | grep -c CLOUDFLARE_API_TOKEN
	```

	* [ ] The command should return `0`

	<br>

	```bash
	sudo cat /proc/$(systemctl show caddy -p MainPID --value)/environ | tr '\0' '\n' | grep CLOUDFLARE
	```

	* [ ] The command should return `CLOUDFLARE_API_TOKEN=<YOUR_CLOUDFLARE_API_TOKEN>`

<br>

### V.&ensp;Go to production

1. In `/etc/caddy/Caddyfile`, remove this line:

	```caddyfile
	acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
	```

2. Check the syntax:

	```bash
	sudo caddy fmt --overwrite /etc/caddy/Caddyfile
	sudo caddy adapt --config /etc/caddy/Caddyfile > /dev/null && echo "Syntax OK"
	```

	* [ ] The last command should return `Syntax OK`

3. Restart **[Caddy](https://caddyserver.com/)**:

	```bash
	sudo systemctl stop caddy
	sudo rm -rf /var/lib/caddy/.local/share/caddy/certificates/acme-staging-v02.api.letsencrypt.org-directory
	sudo systemctl start caddy
	sudo journalctl -u caddy -f | grep -v http.log.access
	```

	* [ ] The last command should show `certificate obtained successfully`

	<br>

	```bash
	sudo ls /var/lib/caddy/.local/share/caddy/certificates/
	```

	* [ ] The name of the file should NOT include `staging`

<br>

### VI.&ensp;Verifications

1. On your local computer:

	```bash
	curl https://tv.<YOUR_DOMAIN>
	curl https://request.<YOUR_DOMAIN>
	```

	* [ ] Both commands should return `tv ok` and `request ok` respectively

	<br>

	```bash
	curl -I http://tv.<YOUR_DOMAIN>
	```

	* [ ] The command should return a `308 Permanent Redirect` to `https://tv.<YOUR_DOMAIN>`

	<br>

	```bash
	curl -sv https://random-test.<YOUR_DOMAIN>
	```

	* [ ] The command should return an error

	<br>

	```bash
	openssl s_client -connect tv.<YOUR_DOMAIN>:443 -servername tv.<YOUR_DOMAIN> </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -ext subjectAltName
	```

	* [ ] The command should NOT show `staging`, and you should see `*.<YOUR_DOMAIN>`, not `tv.<YOUR_DOMAIN>` or `request.<YOUR_DOMAIN>`

2. Reboot the VPS (`sudo reboot`), and redo the first `curl` commands at the beginning of step **1.**

<br>

### VII.&ensp;End-to-end test

1. On the VPS, in `/etc/caddy/Caddyfile`, replace these lines:

	```caddyfile
	@tv host tv.<YOUR_DOMAIN>
	handle @tv {
		respond "tv ok" 200
	}
	```

	with:

	```caddyfile
	@tv host tv.<YOUR_DOMAIN>
	handle @tv {
		reverse_proxy 10.10.0.2:8096
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

4. On the local computer, run a simple Docker container:

	```bash
	docker run -d --name proxytest -p 8096:80 nginx:alpine
	```

5. On your local computer, check that the container is reachable through the VPS:

	```bash
	curl https://tv.<YOUR_DOMAIN>
	```

	* [ ] The command should return the HTML of the Nginx welcome page

6. On the VPS, in `/etc/caddy/Caddyfile`, change the line back to:

	```caddyfile
	@tv host tv.<YOUR_DOMAIN>
	handle @tv {
		respond "tv ok" 200
	}
	```

7. Check the syntax:

	```bash
	sudo caddy fmt --overwrite /etc/caddy/Caddyfile
	sudo caddy adapt --config /etc/caddy/Caddyfile > /dev/null && echo "Syntax OK"
	```

	* [ ] The last command should return `Syntax OK`

8. Reload **[Caddy](https://caddyserver.com/)**:

	```bash
	sudo systemctl reload caddy
	sudo systemctl status caddy --no-pager
	```

	* [ ] The last command should show `active (running)` and no errors

9. On the local computer, clean up the test container:

	```bash
	docker rm -f proxytest
	```

<br>

### [Part 6: Entry point hardening](./06_entry_point_hardening.md)
