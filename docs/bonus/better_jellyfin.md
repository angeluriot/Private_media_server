# Better [Jellyfin](https://jellyfin.org/)

> ⚠️ You may want to apply the updates from **[Part 16: Updates](./16_updates.md)** to your containers before this part

### I.&ensp;[Jellyfin Enhanced](https://github.com/n00bcodr/Jellyfin-Enhanced)

1. In **[Jellyfin](https://jellyfin.org/)**, go to `Dashboard → Plugins → Repositories → +`:

	| Field | Value |
	| --- | --- |
	| Name | `Jellyfin Enhanced` |
	| URL | `https://raw.githubusercontent.com/n00bcodr/jellyfin-plugins/main/manifest.json` |

2. In `Dashboard`, click on `Restart`

3. in `Dashboard → Jellyfin Enhanced`, you can then customize the various options to see what you want for your server

4. Anywhere in **[Jellyfin](https://jellyfin.org/)**, press `?` to show the **[Jellyfin Enhanced](https://github.com/n00bcodr/Jellyfin-Enhanced)** window, you can then customize the various options to see what you like for your account

	I recommend selecting the best options in `Dashboard → Jellyfin Enhanced` rather than `?`, then clicking on `Dashboard → Jellyfin Enhanced → Overview → Quick Actions → Overwrite everyone's personal settings`, that way, even if they can change them later using `?`, at least the users start with good default settings

<br>

### II.&ensp;Unofficial plugins

1. You can find a lot of unofficial plugins for **[Jellyfin](https://jellyfin.org/)** on this repository: **[awesome-jellyfin](https://github.com/awesome-jellyfin/awesome-jellyfin)** (some of which are dependencies for some **[Jellyfin Enhanced](https://github.com/n00bcodr/Jellyfin-Enhanced)** features)

2. Most are easy to install (similar to **[Jellyfin Enhanced](https://github.com/n00bcodr/Jellyfin-Enhanced)**) just by following their README instructions

3. Here is a list of what I personally installed:

	| Plugin | Description |
	| --- | --- |
	| **[Jellyfin Tweaks](https://github.com/n00bcodr/JellyfinTweaks)** | Forces some default settings for every user (it lets you skip parts of the setup from **[Part 14: Users](../docs/14_users.md)**) |
	| **[File Transformation](https://github.com/IAmParadox27/jellyfin-plugin-file-transformation)** | Dependency of:<ul><li>**[Custom Tabs](https://github.com/IAmParadox27/jellyfin-plugin-custom-tabs)**</li><li>**[Plugin Pages](https://github.com/IAmParadox27/jellyfin-plugin-pages)**</li><li>**[Media Bar](https://github.com/IAmParadox27/jellyfin-plugin-media-bar)**</li><li>**[TheIntroDB](https://github.com/TheIntroDB/jellyfin-plugin)**</li><li>**[Intro Skipper](https://github.com/intro-skipper/intro-skipper)**</li><li>**[In Player Episode Preview](https://github.com/Namo2/InPlayerEpisodePreview)**</li><li>**[Better Home Sections](https://github.com/angeluriot/jellyfin-plugin-better-home-sections)**</li></ul> |
	| **[Custom Tabs](https://github.com/IAmParadox27/jellyfin-plugin-custom-tabs)** | Dependency of:<ul><li>**[Jellyfin Enhanced](https://github.com/n00bcodr/Jellyfin-Enhanced)**</li></ul> |
	| **[Plugin Pages](https://github.com/IAmParadox27/jellyfin-plugin-pages)** | Dependency of:<ul><li>**[Jellyfin Enhanced](https://github.com/n00bcodr/Jellyfin-Enhanced)**</li></ul> |
	| **[Media Bar](https://github.com/IAmParadox27/jellyfin-plugin-media-bar)** | Adds a fullscreen slideshow at the top of the home screen showcasing random items of your libraries |
	| **[TheIntroDB](https://github.com/TheIntroDB/jellyfin-plugin)** | Fetches the intro, recap, credits and preview timestamps of your media from the community database **[TheIntroDB](https://theintrodb.org/)** |
	| **[Intro Skipper](https://github.com/intro-skipper/intro-skipper)** | Analyzes your episodes locally (audio fingerprints, silences and black frames) to detect their intros and credits |
	| **[In Player Episode Preview](https://github.com/Namo2/InPlayerEpisodePreview)** | Adds an episode list to the video player to browse the episodes of the series |

4. I also created some plugins:

	| Plugin | Description |
	| --- | --- |
	| **[Genre Mapper](https://github.com/angeluriot/jellyfin-plugin-genre-mapper)** | Maps the genres of your items to a list of genres you choose, which is useful when the metadata providers don't agree (for example **[TMDb](https://www.themoviedb.org/)** gives translated genres while **[TheTVDB](https://thetvdb.com/)** only has English ones) |
	| **[Better Home Sections](https://github.com/angeluriot/jellyfin-plugin-better-home-sections)** | Replaces the home screen sections with sections configured by the administrator (recently added, genres, studios / networks, collections, etc.) |

<br>

### III.&ensp;Unofficial themes

1. You can find some unofficial themes for **[Jellyfin](https://jellyfin.org/)** on this page: **[awesome-jellyfin/THEMES.md](https://github.com/awesome-jellyfin/awesome-jellyfin/blob/main/THEMES.md)**

2. Most are easy to install just by following their README instructions

3. The one I personally installed is **[ElegantFin for Jellyfin 12](https://github.com/mihaif7/elegantfin-jf12)** (install **[ElegantFin](https://github.com/lscambo13/ElegantFin)** instead if you did not upgrade to **[Jellyfin 12](https://jellyfin.org/posts/jellyfin-release-12.0/)**)
