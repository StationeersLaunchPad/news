# SLP'S News & Migration System

StationeersLaunchPad (from version 0.4.0 onwards!) can fetch a remote XML feed on launch and show notices to players.
This can primarily used to warn about broken Workshop mods and offer migration paths to a new version. 
It is also possible to add simple Info notices in case something with SLP breaks and informing the users is needed.

## Feed location

- Default URL (could be overridden in the BepInEx config under the "News" section):
  `https://raw.githubusercontent.com/StationeersLaunchPad/news/main/news.xml`
- The feed is fetched fresh every launch (NO caching).
- On servers or when `NewsCheckOnStart` is false, the system is skipped entirely.
- If the URL is left empty in config, the built-in default is used

## XML structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<NewsFeed>
  <Entry id="unique-id" type="migration_needed" severity="critical"
         heading="Short title shown to the user">
    <short_description>One-liner shown in the list view.</short_description>
    <long_description>Full text shown in the detail view.
Can contain newlines and the usual formatting tags:

[b]bold[/b]                        (larger text only, no true bold font)
[i]italic[/i]                      (no visible effect - italic font not supported)
[u]underline[/u]
[strike]strikethrough[/strike]
[h1]Big header[/h1]
[h2]Medium header[/h2]
[h3]Small header[/h3]
[url=https://example.com]Clickable link[/url]
[code]monospace block[/code]

[red]red text[/red]
[green]green text[/green]
[blue]blue text[/blue]
[yellow]yellow text[/yellow]
[cyan]cyan text[/cyan]

[list]
[*] bullet one
[*] bullet two
[/list]
</long_description>

    <Trigger match_type="workshop_id" workshop_id="3505169479" />
    <!-- or: match_type="workshop_id_and_version" workshop_id="..." version_below="1.2.3" -->
    <!-- or: match_type="mod_name_and_version" mod_name="Exact Name" version_below="..." -->
    <!-- or: match_type="always"   (unconditional, great for testing) -->

    <Actions>
      <Primary label="Migrate Automatically"
               action="repo_mod_install"
               url="https://raw.githubusercontent.com/Org/Repo/refs/heads/modrepo/modrepo.xml"
               modid="the-mod-id-to-install" />

      <!-- Or for Workshop-based replacements: -->
      <!-- <Primary label="Migrate" action="workshop_mod_install" workshop_id="12345678901234567" /> -->

      <Secondary label="View on GitHub"
                 action="open_url"
                 url="https://github.com/Org/Repo" />
    </Actions>
  </Entry>

  <!-- more Entry elements ... -->
</NewsFeed>
```

## Formatting tag gotchas

- `[b]` - renders at 1.15× scale. There is no bold font loaded / shipped, so text is larger but not heavier.
- `[i]` - has no proper visible effect. Italic rendering requires a separate italic font to be registered; none is currently loaded / shipped.

## Entry fields

| Attribute / Element     | Required | Description |
|-------------------------|----------|-------------|
| `id`                    | yes      | Unique stable identifier. Used for dismissal persistence. |
| `type`                  | yes      | `info`, `warning`, `critical`, `migration_needed`, or `mod_broken`. |
| `severity`              | yes      | `info`, `warning`, or `critical` (affects color and urgency). |
| `heading`               | yes      | Title shown in the popup. |
| `short_description`     | yes      | One-line summary shown in the list when multiple notices exist. |
| `long_description`      | yes      | Full body text (supports the formatting tags listed above). |
| `Trigger`               | yes      | See below. |
| `Actions` / `Primary`   | no       | Main button (usually the migration or "OK" action). |
| `Actions` / `Secondary` | no       | Optional second button. |

## Trigger matching

Only **enabled** mods at load time are considered.

Supported `match_type` values:

- `workshop_id` - matches any version of the given Workshop item.
- `workshop_id_and_version` - matches only if the installed version is **below** `version_below`.
- `mod_name_and_version` - matches by exact `mod_name` (case-insensitive) and version below.
- `always` - always matches (useful for showcase / testing feeds).

Version comparison uses the same semver logic as the rest of SLP (`Version.Compare`).

If a notice is a `migration_needed` or `mod_broken` **and** its primary action is a `repo_mod_install` or `workshop_mod_install`, the system will also check whether the target `modID` / `workshopID` is already installed. If so, the notice is skipped silently assuming the user knows what they are doing / to not interfere with manual migrations.

## Actions

### `repo_mod_install`

```xml
<Primary label="Migrate" action="repo_mod_install"
         url="https://.../modrepo.xml" modid="the-mod-id" />
```

- If the `url` points at a GitHub repo root, SLP normalizes it to the usual `refs/heads/modrepo/modrepo.xml`.
- The repo is added (if not already present).
- The exact `modid` is installed using the normal ModRepos pipeline (latest compatible version).
- After a migration action (repo or workshop) completes, a prominent success/failure result is shown in the detail view. The user clicks "Close" to remove the notice for the current session (it will not re-appear unless the replacement is removed again). Failures keep the notice open so the user can retry or ignore.

### `workshop_mod_install`

```xml
<Primary label="Migrate" action="workshop_mod_install" workshop_id="12345678901234567" />
```

- Subscribes the user to the given Steam Workshop item (by its published file id) and waits for download to complete.
- If the triggering `<Trigger>` matched via `workshop_id`, the old (broken) workshop item is unsubscribed after the new one succeeds.
- Workshop subscriptions are handled directly via the Steamworks UGC API
- New workshop mods discovered after the action are automatically enabled (standard "new mod" behavior).
- The "already migrated" check skips the notice if the target `workshop_id` is already subscribed.
- After a successful install the news notice shows a result banner; the user must click "Close" to ack.

### `open_url`

```xml
<Secondary label="More info" action="open_url" url="https://..." />
```

Opens the URL in the user's browser. The notice remains visible (useful for "view details" buttons).

## Ignoring notices

- Every notice has an **Ignore** button.
- Ignores are stored permanently in the BepInEx config under `NewsDismissedIds` (comma-separated list of IDs).
- `info` notices can also be auto-acknowledged after 10 seconds; they are **not** added to the dismissed list.

## Testing / showcase feeds

A minimal always-on feed for testing the UI looks like this:

```xml
<NewsFeed>
  <Entry id="test-info" type="info" severity="info"
         heading="Info notice">
    <short_description>Auto-dismiss demo</short_description>
    <long_description>This will disappear after 10 seconds.</long_description>
    <Trigger match_type="always" />
  </Entry>

  <Entry id="test-migration" type="migration_needed" severity="critical"
         heading="Migration demo">
    <short_description>Click to test repo install</short_description>
    <long_description>Primary button will add a repo and install a mod.</long_description>
    <Trigger match_type="always" />
    <Actions>
      <Primary label="Install test mod" action="repo_mod_install"
               url="https://raw.githubusercontent.com/StationeersCommunityMods/Manifest/refs/heads/modrepo/modrepo.xml"
               modid="LibConstruct" />
    </Actions>
  </Entry>
</NewsFeed>
```

## Implementation notes / how the system is leveraged

- News data loading is driven from the normal mod search path
- `NewsRunner.GetActiveNotices(ModList)` is the main entrypoint that fetches + matches against the current enabled mods.
- The notice UI is shown (with `LoadStage.News` for status) after the mod list is built but before the autoload countdown.
- After a successful mod install, the in-memory mod list is refreshed immediately so the new repo mod appears before configuration.
- All notices are blocking except `info` (10 s timer).
- The system is completely server-safe (skipped when `Platform.IsServer`).
- We can have any number of entries; the UI shows a list when more than one is active.

This gives maintainers a lightweight way to steer players away from broken Workshop content and generally inform users if something is happening they should know about.
