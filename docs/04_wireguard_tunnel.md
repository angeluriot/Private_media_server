# Part 4: [WireGuard](https://www.wireguard.com/) tunnel

### I.&ensp;Create the keys on the VPS

1. Install **[WireGuard](https://www.wireguard.com/)** on the VPS:

	```bash
	sudo apt update && sudo apt install -y wireguard
	```

2. Generate the keys:

	```bash
	sudo -i
	umask 077
	mkdir -p /etc/wireguard && chmod 700 /etc/wireguard
	cd /etc/wireguard
	wg genkey | tee private.key | wg pubkey > public.key
	wg genpsk > preshared.key
	exit
	```

3. Check the files:

	```bash
	sudo ls -l /etc/wireguard
	```

	* [ ] The command should return 3 files, all with `-rw-------` permissions and owned by `root`

4. Save the keys:

	```bash
	sudo cat /etc/wireguard/private.key
	sudo cat /etc/wireguard/public.key
	sudo cat /etc/wireguard/preshared.key
	```

<br>

### II.&ensp;Create the keys on the local computer

1. Install **[WireGuard](https://www.wireguard.com/)** on the local computer:

	```bash
	sudo apt update && sudo apt install -y wireguard
	```

2. Generate the keys:

	```bash
	sudo -i
	umask 077
	mkdir -p /etc/wireguard && chmod 700 /etc/wireguard
	cd /etc/wireguard
	wg genkey | tee private.key | wg pubkey > public.key
	exit
	```

3. Check the files:

	```bash
	sudo ls -l /etc/wireguard
	```

	* [ ] The command should return 2 files, both with `-rw-------` permissions and owned by `root`

4. Save the keys:

	```bash
	sudo cat /etc/wireguard/private.key
	sudo cat /etc/wireguard/public.key
	```

<br>

### III.&ensp;Start [WireGuard](https://www.wireguard.com/) on the VPS

1. On the VPS, create the configuration file:

	```bash
	sudo -i
	umask 077
	nano /etc/wireguard/wg0.conf
	```

	Write the following content in the file:

	```ini
	[Interface]
	Address = 10.10.0.1/24
	ListenPort = 51820
	PrivateKey = <VPS_PRIVATE_KEY>
	MTU = 1280

	[Peer]
	# Local computer
	PublicKey = <LOCAL_PUBLIC_KEY>
	PresharedKey = <VPS_PRESHARED_KEY>
	AllowedIPs = 10.10.0.2/32
	```

	Then:

	```bash
	chmod 600 /etc/wireguard/wg0.conf
	exit
	```

2. Open the port in **[UFW](https://wiki.ubuntu.com/UncomplicatedFirewall)**:

	```bash
	sudo ufw allow 51820/udp comment 'WireGuard'
	sudo ufw status numbered
	```

	* [ ] New `51820/udp` entries should be listed in the output of the last command

3. Start the tunnel:

	```bash
	sudo systemctl enable --now wg-quick@wg0
	sudo systemctl status wg-quick@wg0
	```

	* [ ] The last command should show `active (exited)` and no errors

4. Check the configuration:

	```bash
	sudo wg show
	```

	* [ ] The output should be:

	<br>

	```
	interface: wg0
		public key: <VPS_PUBLIC_KEY>
		private key: (hidden)
		listening port: 51820

	peer: <LOCAL_PUBLIC_KEY>
		preshared key: (hidden)
		allowed ips: 10.10.0.2/32
	```

<br>

### IV.&ensp;Start [WireGuard](https://www.wireguard.com/) on the local computer

1. On the local computer, create the configuration file:

	```bash
	sudo -i
	umask 077
	nano /etc/wireguard/wg0.conf
	```

	Write the following content in the file:

	```ini
	[Interface]
	Address = 10.10.0.2/24
	PrivateKey = <LOCAL_PRIVATE_KEY>
	MTU = 1280

	[Peer]
	# VPS
	PublicKey = <VPS_PUBLIC_KEY>
	PresharedKey = <VPS_PRESHARED_KEY>
	Endpoint = <YOUR_VPS_IP>:51820
	AllowedIPs = 10.10.0.1/32
	PersistentKeepalive = 25
	```

	Then:

	```bash
	chmod 600 /etc/wireguard/wg0.conf
	exit
	```

2. Start the tunnel:

	```bash
	sudo systemctl enable --now wg-quick@wg0
	sudo systemctl status wg-quick@wg0
	```

	* [ ] The last command should show `active (exited)` and no errors

3. Check the interface:

	```bash
	ip addr show wg0
	```

	* [ ] The command should show `mtu 1280` and `inet 10.10.0.2/24`

4. Check the tunnel:

	```bash
	sudo wg show
	```

	* [ ] The command should show a recent `latest handshake` and non-zero `transfer` values

<br>

### V.&ensp;Verify the tunnel

1. Check the tunnel on the VPS:

	```bash
	sudo wg show
	```

	* [ ] The command should show a recent `latest handshake` and non-zero `transfer` values

2. Ping from the local computer:

	```bash
	ping -c 4 10.10.0.1
	```

	* [ ] The command should return 4 successful pings

3. Ping from the VPS:

	```bash
	ping -c 4 10.10.0.2
	```

	* [ ] The command should return 4 successful pings

4. Run a simple Docker container on the local computer:

	```bash
	docker run --rm -d --name wgtest -p 8096:80 nginx
	```

5. On the VPS, check that the container is reachable through the tunnel:

	```bash
	curl -I http://10.10.0.2:8096
	```

	* [ ] The command should return a `200 OK` response

	On the local computer:

	```bash
	docker stop wgtest
	```

6. Reboot the local computer and the VPS (`sudo reboot`), then check that the tunnel is still working by redoing the pings from steps **2.** and **3.**

<br>

### [Part 5: Caddy and wildcard TLS](./05_caddy_and_wildcard_tls.md)
