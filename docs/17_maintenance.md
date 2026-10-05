# Part 17: Maintenance

### Summary

| Task | Frequency |
| --- | --- |
| **[I. Extend the Proton VPN WireGuard key](#iextend-the-proton-vpn-wireguard-key)** | Every 11 months |
| **[II. Update the containers](#iiupdate-the-containers)** | When **[WUD](https://getwud.app/)** notifies you of a new version |

<br>

### I.&ensp;Extend the [Proton VPN](https://protonvpn.com/) WireGuard key

1. Every 11 months, in **[Proton VPN](https://account.proton.me/u/1/vpn)**, go to `WireGuard → mediabox-gluetun` and click on `Extend`

<br>

### II.&ensp;Update the containers

1. When **[What's Up Docker](https://getwud.app/)** notifies you of a new version, open the release notes link of the message and read the notes of every version between yours and the new one, and look for breaking changes or manual steps (an LLM can greatly help you with this)

2. On the local computer, run the update command given in the message:

	```bash
	sudo mediaserver-update <SERVICE> <NEW_TAG>
	```

	* [ ] The last line should start with `✅`

	* [ ] The web interface of the service should work and show the new version (usually in `System → Status` or `Dashboard`)

3. If something is wrong, go back to the previous version:

	```bash
	sudo mediaserver-update --rollback <SERVICE>
	```
