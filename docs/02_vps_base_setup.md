# Part 2: VPS base setup

### I.&ensp;Initialize the VPS

1. On the VPS, update the system:

	```bash
	sudo apt update && sudo apt full-upgrade -y
	sudo apt autoremove -y
	```

2. Set the time zone:

	```bash
	sudo timedatectl set-timezone <YOUR_TIMEZONE>
	timedatectl
	```

	* [ ] The time zone should be correct and the system clock should be synchronized

3. Set the hostname:

	```bash
	sudo hostnamectl set-hostname edge-vps
	sudo sed -i "s/^127.0.1.1.*/127.0.1.1\tedge-vps/" /etc/hosts
	echo "preserve_hostname: true" | sudo tee /etc/cloud/cloud.cfg.d/99-hostname.cfg
	grep 127.0.1.1 /etc/hosts
	```

	* [ ] The last command should return `127.0.1.1 edge-vps`

<br>

### II.&ensp;Harden SSH

1. Create an SSH key pair on your local computer and copy the public key to the VPS:

	```bash
	ssh-keygen -t ed25519 -C "<YOUR_LOCAL_USER>@<YOUR_LOCAL_HOSTNAME>"
	ssh-copy-id -i ~/.ssh/id_ed25519.pub <YOUR_VPS_USER>@<YOUR_VPS_IP>
	```

2. On your local computer:

	```bash
	ssh <YOUR_VPS_USER>@<YOUR_VPS_IP>
	```

	* [ ] You should be able to log in without a password prompt (not counting the passphrase for the SSH key)

3. On the VPS, disable password authentication and root login:

	```bash
	sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null <<'EOF'
	PasswordAuthentication no
	KbdInteractiveAuthentication no
	PermitRootLogin no
	PubkeyAuthentication yes
	X11Forwarding no
	MaxAuthTries 3
	LoginGraceTime 20
	AllowUsers <YOUR_VPS_USER>
	EOF
	```

	```bash
	sudo chmod 600 /etc/ssh/sshd_config.d/10-hardening.conf
	sudo sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config.d/50-cloud-init.conf
	```

4. Check the VPS SSH configuration:

	```bash
	sudo sshd -t && echo OK
	```

	* [ ] The command should return `OK`

	<br>

	```bash
	sudo sshd -T | grep -E "^(passwordauthentication|permitrootlogin|pubkeyauthentication|kbdinteractiveauthentication|maxauthtries)"
	```

	* [ ] The command should return:

	<br>

	```
	maxauthtries 3
	permitrootlogin no
	pubkeyauthentication yes
	passwordauthentication no
	kbdinteractiveauthentication no
	```

5. Restart the VPS SSH service:

	```bash
	sudo systemctl restart ssh
	```

6. On your local computer:

	```bash
	ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <YOUR_VPS_USER>@<YOUR_VPS_IP>
	```

	* [ ] You should get `Permission denied (publickey)`

<br>

### III.&ensp;Reduce listening services

1. On the VPS, disable LLMNR:

	```bash
	sudo mkdir -p /etc/systemd/resolved.conf.d
	```

	```bash
	sudo tee /etc/systemd/resolved.conf.d/99-disable-llmnr.conf > /dev/null <<'EOF'
	[Resolve]
	LLMNR=no
	MulticastDNS=no
	EOF
	```

	```bash
	sudo systemctl restart systemd-resolved
	```

2. Check the listening services on the VPS:

	```bash
	ss -tulpn
	```

	* [ ] No `5355` port should be visible

<br>

### IV.&ensp;Firewall

1. Install **[UFW](https://wiki.ubuntu.com/UncomplicatedFirewall)** on the VPS:

	```bash
	sudo apt update && sudo apt install -y ufw
	```

2. Check that IPv6 is enabled:

	```bash
	cat /etc/default/ufw | grep IPV6
	```

	* [ ] The command should return `IPV6=yes`

3. Only allow port 22 (SSH) for now:

	```bash
	sudo ufw default deny incoming
	sudo ufw default allow outgoing
	sudo ufw limit 22/tcp comment 'SSH'
	sudo ufw enable
	```

4. Verification:

	```bash
	sudo ufw status verbose
	```

	* [ ] The command should return `Status: active` and only SSH rules

<br>

### V.&ensp;Automatic updates

1. Install **[unattended-upgrades](https://wiki.debian.org/UnattendedUpgrades)** on the VPS:

	```bash
	sudo apt update && sudo apt install -y unattended-upgrades apt-listchanges
	sudo dpkg-reconfigure -plow unattended-upgrades
	```

2. Open `/etc/apt/apt.conf.d/50unattended-upgrades`:

	Uncomment these 3 lines:

	```
	...
	Unattended-Upgrade::Origins-Pattern {
		...
		"origin=Debian,codename=${distro_codename}-updates";
		...
		"origin=Debian,codename=${distro_codename},label=Debian-Security";
		"origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
		...
	};
	...
	```

	Uncomment these lines and set them to the following values (except for the `"05:00"`, it's just an example, set it to a time you are most likely not watching anything):

	```
	...
	Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
	...
	Unattended-Upgrade::Automatic-Reboot "true";
	...
	Unattended-Upgrade::Automatic-Reboot-Time "05:00";
	...
	```

3. Check that **[unattended-upgrades](https://wiki.debian.org/UnattendedUpgrades)** is working:

	```bash
	sudo unattended-upgrades --dry-run --debug
	```

	* [ ] The command shouldn't return any error

	<br>

	```bash
	systemctl is-enabled unattended-upgrades
	```

	* [ ] The command should return `enabled`

	<br>

	```bash
	systemctl list-timers | grep apt
	```

	* [ ] `apt-daily-upgrade` and `apt-daily` should be listed

<br>

### VI.&ensp;Final verification

1. Restart the VPS:

	```bash
	sudo reboot
	```

	Wait a few seconds, then reconnect with SSH and check that the reboot was successful:

	```bash
	uptime -p
	systemctl is-system-running
	```

	* [ ] The first command should return a short uptime like `up 1 minute`, and the second command should return `running`

2. Check that everything is still working:

	```bash
	hostnamectl | grep -i "static hostname"
	```

	* [ ] The command should return `Static hostname: edge-vps`

	<br>

	```bash
	timedatectl
	```

	* [ ] The time zone should be correct and the system clock should be synchronized

	<br>

	```bash
	sudo sshd -T | grep -E "^(passwordauthentication|permitrootlogin|pubkeyauthentication|kbdinteractiveauthentication|maxauthtries)"
	```

	* [ ] The command should return:

	<br>

	```
	maxauthtries 3
	permitrootlogin no
	pubkeyauthentication yes
	passwordauthentication no
	kbdinteractiveauthentication no
	```

	<br>

	```bash
	ss -tulpn | grep -v 127.0.0
	```

	* [ ] No `5355` port should be visible

	<br>

	```bash
	sudo ufw status verbose
	```

	* [ ] The command should return `Status: active` and only SSH rules

	<br>

	```bash
	systemctl is-enabled unattended-upgrades
	systemctl list-timers apt-daily-upgrade.timer
	```

	* [ ] `enabled` and a planned next run should be listed

<br>

### [Part 3: Local computer base setup](./03_local_computer_base_setup.md)
