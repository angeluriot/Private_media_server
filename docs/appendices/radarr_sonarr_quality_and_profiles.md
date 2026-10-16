# [Radarr](https://radarr.video/) / [Sonarr](https://sonarr.tv/) quality and profiles

The quality and profiles I personally used for **[Radarr](https://radarr.video/)** and **[Sonarr](https://sonarr.tv/)**, as a French speaker watching original and French-dubbed 1080p content (adapt them to your own needs):

<br>

### I.&ensp;Quality

1. Go to `Settings → Quality`:

	| Field | Min | Preferred | Max |
	| --- | --- | --- | --- |
	| `*-720p` | `5` | `30` | `50` |
	| `*-1080p` | `10` | `40` | `90` |

	Save

<br>

### II.&ensp;Custom formats

1. Go to `Settings → Custom Formats → +`, `Multi audio` as Name, then `Conditions → + → Release Title`:

	| Field | Value |
	| --- | --- |
	| Name | `Multi-audio-tags` |
	| Negate | ❌ |
	| Required | ✅ |

	Regular Expression:

	```regex
	(?i)\bMULTI(?![\s._-]?SUB)\b
	```

	Save

2. Create another custom format, `Multi subs` as Name, then `Conditions → + → Release Title`:

	| Field | Value |
	| --- | --- |
	| Name | `Multi-subs-tags` |
	| Negate | ❌ |
	| Required | ✅ |

	Regular Expression:

	```regex
	(?i)\b(?:MULTI|MULTIPLE|DUAL)[\s._-]?SUB(?:S|BED|TITLES?)?\b
	```

	Save

3. Create another custom format, `VF` as Name, then `Conditions → + → Language`:

	| Field | Value |
	| --- | --- |
	| Name | `French` |
	| Language | `French` |
	| Except Language | ❌ |
	| Negate | ❌ |
	| Required | ❌ |

	Then `Conditions → + → Release Title`:

	| Field | Value |
	| --- | --- |
	| Name | `VF-tags` |
	| Negate | ❌ |
	| Required | ❌ |

	Regular Expression:

	```regex
	(?i)\b(?:TRUEFRENCH|VF[FQI2]?)\b
	```

	Save

4. Create another custom format, `VO` as Name, then `Conditions → + → Language`:

	| Field | Value |
	| --- | --- |
	| Name | `Original` |
	| Language | `Original` |
	| Except Language | ❌ |
	| Negate | ❌ |
	| Required | ✅ |

	Save

5. Create another custom format, `VOSTFR` as Name, then `Conditions → + → Release Title`:

	| Field | Value |
	| --- | --- |
	| Name | `VOSTFR-tags` |
	| Negate | ❌ |
	| Required | ✅ |

	Regular Expression:

	```regex
	(?i)\b(?:VOSTFR|VOST[\s._-]?FR(?:ENCH)?|SUB[\s._-]?FR(?:ENCH|A|E)?|FR(?:ENCH|A|E)?[\s._-]?SUB(?:S|BED|TITLES?)?|ST[\s._-]?FR|FR[\s._-]?ST)\b
	```

	Save

6. Create another custom format, `Unwanted` as Name, then `Conditions → + → Release Title`:

	| Field | Value |
	| --- | --- |
	| Name | `BR-DISK` |
	| Negate | ❌ |
	| Required | ❌ |

	Regular Expression:

	```regex
	^(?!.*\b((?<!HD[._ -]|HD)DVD|BDRip|MKV|XviD|WMV|d3g|(BD)?REMUX|^(?=.*1080p)(?=.*HEVC)|[xh][-_. ]?26[45]|German.*[DM]L|((?<=\d{4}).*German.*([DM]L)?)(?=.*\b(AVC|HEVC|VC[-_. ]?1|MVC|MPEG[-_. ]?2)\b))\b)(((?=.*\b(Blu[-_. ]?ray|BD|HD[-_. ]?DVD)\b)(?=.*\b(AVC|HEVC|VC[-_. ]?1|MVC|MPEG[-_. ]?2|BDMV|ISO)\b))|^((?=.*\b(((?=.*\b((.*_)?COMPLETE.*|Dis[ck])\b)(?=.*(Blu[-_. ]?ray|HD[-_. ]?DVD)))|3D[-_. ]?BD|BR[-_. ]?DISK|Full[-_. ]?Blu[-_. ]?ray|^((?=.*((BD|UHD)[-_. ]?(25|50|66|100|ISO)))))))).*
	```

	Then `Conditions → + → Release Title` (**[Radarr](https://radarr.video/)** only):

	| Field | Value |
	| --- | --- |
	| Name | `3D` |
	| Negate | ❌ |
	| Required | ❌ |

	Regular Expression:

	```regex
	(?i)(?<=\b[12]\d{3}\b).*\b(?:3d|sbs|half[ .-]ou|half[ .-]sbs)\b|\b(?:BluRay3D|BD3D)\b
	```

	Then `Conditions → + → Release Title`:

	| Field | Value |
	| --- | --- |
	| Name | `Extras` |
	| Negate | ❌ |
	| Required | ❌ |

	Regular Expression for **[Radarr](https://radarr.video/)**:

	```regex
	(?<=\b[12]\d{3}\b).*\b(Extras|Bonus|Extended[ ._-]Clip)\b
	```

	Regular Expression for **[Sonarr](https://sonarr.tv/)**:

	```regex
	(?<=\bS\d+\b).*\b(Extras|Bonus|Extended[ ._-]Clip)\b
	```

	Save

7. Create another custom format, `Bad subtitles` as Name, then `Conditions → + → Release Title`:

	| Field | Value |
	| --- | --- |
	| Name | `FastSUB` |
	| Negate | ❌ |
	| Required | ❌ |

	Regular Expression:

	```regex
	(?i)\bFastSUB\b
	```

	Then `Conditions → + → Release Title`:

	| Field | Value |
	| --- | --- |
	| Name | `Anime-raws` |
	| Negate | ❌ |
	| Required | ❌ |

	Regular Expression:

	```regex
	(?i)(?:Asuka|Beatrice|Daddy|Fumi|Iriza|Kawaiika|Koi|Lilith|LowPower|Nanako|NC|neko|New|Ohys|Pandoratv|Scryous|Seicher|Shiniori)[ ._-]?Raws|\[km\]|-km\b|\b(?:Moozzi2|Raws-Maji|ReinForce)\b
	```

	Save

8. Reload both pages

<br>

### III.&ensp;Profiles

1. Go to `Settings → Profiles → +`:

	Click on "Edit Groups" and create these 2 groups:

	| Group | Qualities |
	| --- | --- |
	| `1080p` | `Bluray-1080p` `WEBRip-1080p` `WEBDL-1080p` |
	| `720p` | `Bluray-720p` `WEBRip-720p` `WEBDL-720p` |

	Click on "Done Editing Groups" and move the elements of the list like this:

	| Rank | Name | Selected |
	| --- | --- | --- |
	| **1** | `1080p` | ✅ |
	| **2** | `HDTV-1080p` | ✅ |
	| **3** | `720p` | ✅ |
	| **4** | `HDTV-720p` | ✅ |
	| **...** | Everything else | ❌ |

	Then, on the left of the window:

	| Field | Value |
	| --- | --- |
	| Name | `VF 1080p` |
	| Upgrades Allowed | ✅ |
	| Upgrade Until | `1080p` |
	| Minimum Custom Format Score | `20` |
	| Upgrade Until Custom Format Score | `1000` |
	| Minimum Custom Format Score Increment | `1` |
	| Language | `Any` |

	| Custom Format | Score |
	| --- | --- |
	| `VF` | `100` |
	| `Multi audio` | `20` |
	| `VO` | `10` |
	| `Multi subs` | `2` |
	| `VOSTFR` | `2` |
	| `Bad subtitles` | `-5` |
	| `Unwanted` | `-1000` |

	Save

2. Create another profile `VO 1080p` with the same settings (you can use the "Clone Profile" button) except:

	| Custom Format | Score |
	| --- | --- |
	| `VOSTFR` | `100` |
	| `VO` | `50` |
	| `Multi subs` | `20` |
	| `Multi audio` | `2` |
	| `VF` | `2` |

	Save

3. Go to `Movies`/`Series` and `Collections`, and change the profile of each element to the new profiles you created

4. Delete the old profiles
