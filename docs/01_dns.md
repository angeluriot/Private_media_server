# Part 1: DNS

### I.&ensp;Add 4 DNS records for your domain on [Cloudflare](https://www.cloudflare.com/)

1. Go to `Domains → Overview → <YOUR_DOMAIN> → DNS → Records → Add record`

2. Add the following records:

	| Type | Name | Value | Proxy status | TTL |
	| --- | --- | --- | --- | --- |
	| A | `tv` | \<Your VPS IPv4 address\> | DNS only | Auto |
	| A | `request` | \<Your VPS IPv4 address\> | DNS only | Auto |
	| CAA | `@` | `0 issue "letsencrypt.org"` | DNS only | Auto |
	| CAA | `@` | `0 issuewild "letsencrypt.org"` | DNS only | Auto |

3. On your local computer:

	```bash
	dig +short tv.<YOUR_DOMAIN>
	dig +short tv.<YOUR_DOMAIN> @1.1.1.1
	dig +short tv.<YOUR_DOMAIN> @8.8.8.8
	dig +short request.<YOUR_DOMAIN>
	dig +short request.<YOUR_DOMAIN> @1.1.1.1
	dig +short request.<YOUR_DOMAIN> @8.8.8.8
	```

	* [ ] All commands should return the VPS IPv4 address

	<br>

	```bash
	dig +short random-test.<YOUR_DOMAIN>
	```

	* [ ] The command should return nothing, if it returns an IP address, remove the wildcard DNS record causing it

<br>

### II.&ensp;Create a [Cloudflare](https://www.cloudflare.com/) API token

1. Go to `My Profile → API Tokens → Create Token → Create Custom Token`

2. Set the following values:

	| Field | Value |
	| --- | --- |
	| Permission | `Zone → Zone → Read` |
	| Permission | `Zone → DNS → Edit` |
	| Zone Resources | `Include → Specific zone → <YOUR_DOMAIN>` |
	| Client IP Filtering | \<Your VPS IPv4 + IPv6 addresses\> (optional) |
	| TTL | `No expiration` |

3. Save the API token

4. On your VPS (or anywhere if you didn't set the Client IP Filtering):

	```bash
	curl -s -H "Authorization: Bearer <YOUR_CLOUDFLARE_API_TOKEN>" https://api.cloudflare.com/client/v4/user/tokens/verify
	```

	* [ ] The command should return a JSON object with `"success": true`

	<br>

	```bash
	curl -s -H "Authorization: Bearer <YOUR_CLOUDFLARE_API_TOKEN>" "https://api.cloudflare.com/client/v4/zones?name=<YOUR_DOMAIN>" | grep -o '"id":"[^"]*"' | head -1
	```

	* [ ] The command should return an ID

<br>

### [Part 2: VPS base setup](./02_vps_base_setup.md)
