# Part 3: Local computer base setup

### I.&ensp;Initialize the local computer

1. Update the system:

	```bash
	sudo apt update && sudo apt full-upgrade -y
	sudo apt autoremove -y
	```

2. Check the UID and GID:

	```bash
	id -u
	id -g
	```

	* [ ] Both should be `1000`

3. Set the default file creation permissions:

	```bash
	echo "umask 002" | sudo tee /etc/profile.d/umask.sh
	```

4. Set the time zone:

	```bash
	sudo timedatectl set-timezone <YOUR_TIMEZONE>
	timedatectl
	```

	* [ ] The time zone should be correct and the system clock should be synchronized

5. Set the hostname (if not already set):

	```bash
	sudo hostnamectl set-hostname mediabox
	sudo sed -i "s/^127.0.1.1.*/127.0.1.1\tmediabox/" /etc/hosts
	```

6. Check the hostname:

	```bash
	hostnamectl | grep -i "static hostname"
	```

	* [ ] The command should return `Static hostname: mediabox`

	<br>

	```bash
	hostname
	```

	* [ ] The command should return `mediabox`

	<br>

	```bash
	grep mediabox /etc/hosts
	```

	* [ ] The command should return a line with `mediabox` in it

<br>

### II.&ensp;Storage

1. Identify the disks:

	```bash
	lsblk -o NAME,SIZE,ROTA,TYPE,FSTYPE,MOUNTPOINTS,MODEL
	```

	* [ ] You should see the SSD (`ROTA` = `0`) holding the system partitions (`/`, `/boot/efi`, ...) and the hard drive (`ROTA` = `1`), remember its name

2. Check the health of the hard drive:

	```bash
	sudo apt update && sudo apt install -y smartmontools
	sudo smartctl -H /dev/<YOUR_HDD_NAME>
	```

	* [ ] The last command should return a `PASSED` result

3. Format the hard drive (only if it's a new one, it will erase all existing data):

	```bash
	sudo apt update && sudo apt install -y parted
	sudo wipefs -a /dev/<YOUR_HDD_NAME>
	sudo parted -s /dev/<YOUR_HDD_NAME> mklabel gpt mkpart data ext4 0% 100%
	sudo mkfs.ext4 -m 0 -L data /dev/<YOUR_HDD_NAME>1
	lsblk -o NAME,SIZE,FSTYPE,LABEL /dev/<YOUR_HDD_NAME>
	```

	* [ ] The last command should show a single `<YOUR_HDD_NAME>1` partition with `ext4` as `FSTYPE` and `data` as `LABEL`

4. Mount the hard drive on `/data`:

	```bash
	sudo mkdir -p /data
	echo "UUID=$(sudo blkid -s UUID -o value /dev/<YOUR_HDD_NAME>1) /data ext4 defaults,noatime,nofail 0 2" | sudo tee -a /etc/fstab
	```

	* [ ] The line printed by the last command should have a UUID after `UUID=`

	<br>

	```bash
	sudo systemctl daemon-reload
	sudo mount -a
	findmnt /data
	```

	* [ ] The last command should show `/dev/<YOUR_HDD_NAME>1` as `SOURCE` and `ext4` as `FSTYPE`

	<br>

	```bash
	df -h /data
	```

	* [ ] The command should show the size of the hard drive

5. Create the directories where the content and configuration will be stored:

	```bash
	sudo mkdir -p /data/torrents/{incomplete,movies,tv,anime}
	sudo mkdir -p /data/media/{movies,tv,anime}
	sudo mkdir -p /opt/mediaserver/config/{jellyfin,seerr,radarr,sonarr,prowlarr,bazarr,qbittorrent,gluetun}
	sudo mkdir -p /opt/mediaserver/cache/jellyfin
	sudo chown -R 1000:1000 /data /opt/mediaserver
	sudo chmod -R 775 /data
	sudo find /data -type d -exec chmod g+s {} \;
	ls -la /data/torrents /data/media
	```

	* [ ] All directories shown by the last command should be owned by `1000` `1000` (or your user) and have permissions `drwxrwsr-x`

6. Check that both directories are mounted on the same disk:

	```bash
	stat -c '%d %n' /data/torrents /data/media
	```

	* [ ] Both numbers should be the same

	<br>

	```bash
	echo "test" > /data/torrents/hardlink-test
	ln /data/torrents/hardlink-test /data/media/hardlink-test
	stat -c '%h %i %n' /data/torrents/hardlink-test /data/media/hardlink-test
	```

	* [ ] Both lines should start with `2` then another number, it should be the same number for both lines

	<br>

	```bash
	rm /data/torrents/hardlink-test /data/media/hardlink-test
	```

<br>

### III.&ensp;Docker

1. Remove the current Docker installation (if any):

	```bash
	for pkg in docker.io docker-compose docker-doc docker-buildx podman-docker containerd runc; do
	  sudo apt remove -y $pkg 2>/dev/null
	done
	```

2. Install Docker:

	```bash
	sudo apt update && sudo apt install -y ca-certificates curl
	sudo install -m 0755 -d /etc/apt/keyrings
	sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
	sudo chmod a+r /etc/apt/keyrings/docker.asc
	```

	```bash
	sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
	Types: deb
	URIs: https://download.docker.com/linux/debian
	Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
	Components: stable
	Architectures: $(dpkg --print-architecture)
	Signed-By: /etc/apt/keyrings/docker.asc
	EOF
	```

	```bash
	sudo apt update
	apt-cache policy docker-ce | grep -m1 "download.docker.com"
	```

	* [ ] The last command should return a line with `download.docker.com` in it

	<br>

	```bash
	sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
	```

3. Configure the daemon:

	```bash
	sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
	{
	  "dns": ["1.1.1.1", "9.9.9.9"],
	  "log-driver": "json-file",
	  "log-opts": {
	    "max-size": "10m",
	    "max-file": "3"
	  }
	}
	EOF
	```

	Prevent Docker from starting if the hard drive isn't mounted:

	```bash
	sudo mkdir -p /etc/systemd/system/docker.service.d
	```

	```bash
	sudo tee /etc/systemd/system/docker.service.d/10-require-data.conf > /dev/null <<'EOF'
	[Unit]
	RequiresMountsFor=/data
	EOF
	```

	```bash
	sudo systemctl daemon-reload
	sudo systemctl enable --now docker
	sudo systemctl restart docker
	```

4. Put the user in the docker group:

	```bash
	sudo usermod -aG docker $USER
	```

	Then close the terminal and open a new one

5. Check that Docker is working:

	```bash
	docker run --rm hello-world
	```

	* [ ] The command should return a message saying `Hello from Docker!`

	<br>

	```bash
	docker version
	docker compose version
	docker info | grep -E "Storage Driver|Logging Driver|Cgroup"
	```

	* [ ] The first two commands should return the version of Docker and Docker Compose, the last should show `overlayfs` or `overlay2` as `Storage Driver`

	<br>

	```bash
	docker run --rm alpine nslookup discord.com
	```

	* [ ] The command should return IP addresses for `discord.com`

	<br>

	```bash
	systemctl is-enabled docker
	```

	* [ ] The command should return `enabled`

	<br>

	```bash
	systemctl show docker -p RequiresMountsFor
	```

	* [ ] The command should return `RequiresMountsFor=/data`

<br>

### IV.&ensp;Swap

1. Check the current swap:

	```bash
	sudo swapon --show
	free -h
	```

	If the first command lists a line with ≈8 GB and the second lists a `Swap` line with ≈8 GB, skip to step **3.**

2. Create an 8 GB swap file on the SSD:

	```bash
	sudo fallocate -l 8G /swapfile
	sudo chmod 600 /swapfile
	sudo mkswap /swapfile
	sudo swapon /swapfile
	echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab
	```

3. Only use the swap when the RAM is really full:

	```bash
	echo "vm.swappiness=10" | sudo tee /etc/sysctl.d/99-swappiness.conf
	sudo sysctl --system
	```

4. Check the swap:

	```bash
	sudo swapon --show
	```

	* [ ] The command should list a swap of about 8 GB

	<br>

	```bash
	cat /proc/sys/vm/swappiness
	```

	* [ ] The command should return `10`

<br>

### V.&ensp;Transcoding hardware setup

1. Check that the **[Intel](https://www.intel.com/)** **[iGPU](https://en.wikipedia.org/wiki/List_of_Intel_graphics_processing_units)** is detected:

	```bash
	lspci -nn | grep -iE "vga|display"
	```

	* [ ] The command should return a line with `Intel` and `Graphics` in it

	<br>

	```bash
	ls -l /dev/dri
	```

	* [ ] The command should list a `renderD128` device belonging to the `render` group

2. Install the GPU firmware, then reboot:

	```bash
	sudo apt update && sudo apt install -y firmware-misc-nonfree
	sudo reboot
	```

3. Check that the GuC and HuC firmware are loaded:

	```bash
	sudo dmesg | grep -iE "guc|huc"
	```

	* [ ] A line about `HuC` should contain `authenticated`

4. Get the GID of the `render` group:

	```bash
	getent group render | cut -d: -f3
	```

	Save this number

5. Check that the GPU is usable from a **[Jellyfin](https://jellyfin.org/)** container with the same user it will run as:

	```bash
	docker run --rm --user 1000:1000 --group-add <YOUR_RENDER_GID> --device /dev/dri:/dev/dri --entrypoint /usr/lib/jellyfin-ffmpeg/vainfo jellyfin/jellyfin:10.11 --display drm --device /dev/dri/renderD128
	```

	* [ ] The command should show `Intel iHD driver` without any error, followed by a list of `VAProfile...` lines, some of them ending with `VAEntrypointEncSliceLP`

<br>

### VI.&ensp;Always on

1. In the BIOS / UEFI settings of the local computer, find the option usually called `Restore on AC Power Loss` (or `AC Back`, `After Power Failure`, ...) and set it to `Power On`, so the computer restarts by itself after a power outage

2. Disable sleep and hibernation:

	```bash
	sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
	```

3. Check that sleep is disabled:

	```bash
	systemctl is-enabled sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
	```

	* [ ] The command should return `masked` 5 times

<br>

### VII.&ensp;Automatic updates

1. Install **[unattended-upgrades](https://wiki.debian.org/UnattendedUpgrades)**:

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

### VIII.&ensp;Final verification

1. Restart the local computer:

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

	* [ ] The command should return `Static hostname: mediabox`

	<br>

	```bash
	timedatectl
	```

	* [ ] The time zone should be correct and the system clock should be synchronized

	<br>

	```bash
	findmnt /data
	```

	* [ ] The command should show `/dev/<YOUR_HDD_NAME>1` as `SOURCE` and `ext4` as `FSTYPE`

	<br>

	```bash
	sudo swapon --show
	```

	* [ ] The command should list a swap of about 8 GB

	<br>

	```bash
	systemctl is-active docker
	docker run --rm hello-world
	```

	* [ ] The first command should return `active` and the second one `Hello from Docker!`

	<br>

	```bash
	sudo dmesg | grep -i huc
	```

	* [ ] A line should contain `authenticated`

	<br>

	```bash
	systemctl is-enabled sleep.target
	```

	* [ ] The command should return `masked`

	<br>

	```bash
	systemctl is-enabled unattended-upgrades
	systemctl list-timers apt-daily-upgrade.timer
	```

	* [ ] `enabled` and a planned next run should be listed

<br>

### [Part 4: WireGuard tunnel](./04_wireguard_tunnel.md)
