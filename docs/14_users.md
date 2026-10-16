# Part 14: Users

### I.&ensp;Create users

1. You need to create the account for each user, in **[Jellyfin](https://jellyfin.org/)**, go to `Dashboard → Users → +`:

	| Field | Value |
	| --- | --- |
	| Username | `<A_NEW_USERNAME>` |
	| Password | `<A_NEW_PASSWORD>` |
	| Enable access to all libraries | ✅ |

	Save

2. Then on the next screen:

	| Field | Value |
	| --- | --- |
	| Maximum number of simultaneous user sessions | `3` |

	Save

3. Go back to the user, go to `Edit this user's profile, image and personal preferences. → Display`:

	| Field | Value |
	| --- | --- |
	| Display language | `<THEIR_PREFERRED_LANGUAGE>` |
	| Date time locale | `<THEIR_PREFERRED_LANGUAGE>` |

	Save

4. In `Edit this user's profile, image and personal preferences. → Home`, choose a simple default layout like this for example:

	| Field | Value |
	| --- | --- |
	| Home screen section 1 | `My Media` |
	| Home screen section 2 | `Recently Added Media` |
	| Home screen section 3 | `Continue Watching` |
	| Home screen section 4-10 | `None` |

	Library Order: `Movies`, `TV Shows`, `Anime`

	Save

5. In `Edit this user's profile, image and personal preferences. → Playback`:

	| Field | Value |
	| --- | --- |
	| Preferred audio language | `<THEIR_PREFERRED_LANGUAGE>` (can be "Original language") |
	| Play default audio track regardless of language | ❌ |

	Save

6. In `Edit this user's profile, image and personal preferences. → Subtitles`:

	| Field | Value |
	| --- | --- |
	| Preferred subtitle language | `<THEIR_PREFERRED_LANGUAGE>` |
	| Subtitle mode | `Smart` |
	| Text size | `Large` |
	| Drop shadow | `Uniform` |

	Save

<br>

### II.&ensp;Onboarding

1. Give the username and password to the user so they can log in with their account in:

	* `tv.<YOUR_DOMAIN>` on their browser or the **[Jellyfin](https://jellyfin.org/)** mobile / smart TV app

		* They have a limited number of simultaneous sessions as configured earlier

	* `request.<YOUR_DOMAIN>` on their browser or the **[Seerr](https://seerr.dev/)** mobile app

		* They have a limited number of requests per unit of time as configured earlier

<br>

### [Part 15: Monitoring](./15_monitoring.md)
