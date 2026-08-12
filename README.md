# SLP News & Migration System

StationeersLaunchPad (version 0.4.0 and later) can fetch a remote XML feed on launch and show notices to players. Notices can report general information, warn about broken Workshop mods, offer migrations, or let users subscribe to an additional Workshop item.

## Feed location

- Default URL (overridable in the BepInEx config under `News`): `https://raw.githubusercontent.com/StationeersLaunchPad/news/main/news.xml`
- The feed is fetched on every launch; it is not cached.
- The system is skipped on servers and when `NewsCheckOnStart` is false.
- If the configured URL is empty, the built-in default is used.

## Complete example

```xml
<?xml version="1.0" encoding="utf-8"?>
<NewsFeed>
  <Entry id="unique-id" type="migration_needed" severity="critical"
         heading="Short title shown to the user">
    <short_description>One-line summary shown in the notice list.</short_description>
    <long_description>Full text shown in the detail view.</long_description>

    <Trigger match_type="workshop_id"
             workshop_id="3505169479"
             unless_workshop_id="1234567890" />

    <Actions>
      <Primary label="Migrate automatically"
               action="repo_mod_install"
               url="https://raw.githubusercontent.com/Org/Repo/refs/heads/modrepo/modrepo.xml"
               modid="the-mod-id-to-install" />
      <Secondary label="View on GitHub"
                 action="open_url"
                 url="https://github.com/Org/Repo" />
    </Actions>
  </Entry>
</NewsFeed>
```

Use [notices-builder.html](notices-builder.html) to build entries using a simple form and copy or download the resulting XML.

## Entry fields

| Attribute / element | Required | Description |
|---|---:|---|
| `id` | yes | Unique stable identifier used for dismissal persistence. |
| `type` | yes | `info`, `warning`, `critical`, `migration_needed`, or `mod_broken`. |
| `severity` | yes | `info`, `warning`, or `critical`; controls color and urgency. |
| `heading` | yes | Title shown in the popup. |
| `short_description` | yes | One-line summary shown when multiple notices exist. |
| `long_description` | yes | Full notice body; supports the formatting tags below. |
| `Trigger` | yes | Determines when the notice is shown. |
| `Actions` / `Primary` | no | Main button. |
| `Actions` / `Secondary` | no | Optional second button. |

## Trigger matching

Only enabled mods are considered when matching a mod trigger.

| `match_type` | Required attributes | Behavior |
|---|---|---|
| `workshop_id` | `workshop_id` | Matches any version of the Workshop item. |
| `workshop_id_and_version` | `workshop_id`, `version_below` | Matches when the installed version is below the specified version. |
| `mod_name_and_version` | `mod_name`, `version_below` | Matches the exact mod name (case-insensitive) below the specified version. |
| `always` | none | Always matches; useful for announcements and testing. |

Version checks use SLP's normal `Version.Compare` logic. A mod with no version also matches a `version_below` trigger.

### Hide when a Workshop item is installed

Every trigger can include the optional `unless_workshop_id` attribute:

```xml
<Trigger match_type="always" unless_workshop_id="1234567890" />
```

The notice is skipped when that Workshop item is already installed. This is useful for announcements or optional dependencies where the notice should disappear as soon as the user has the relevant item. The exclusion checks all discovered mods, not only enabled mods.

Migration notices (`migration_needed` and `mod_broken`) also have an automatic replacement check when their primary action is `repo_mod_install` or `workshop_mod_install`. They are hidden when that primary replacement is already installed. Use `unless_workshop_id` when you need the same behavior independently of notice type or action placement.

## Actions

Both `<Primary>` and `<Secondary>` accept `label` and `action`. Action-specific attributes are shown below.

### `repo_mod_install`

```xml
<Primary label="Migrate" action="repo_mod_install"
         url="https://example.com/modrepo.xml" modid="the-mod-id" />
```

- Supported on the primary button.
- Adds the repository if necessary and installs the requested `modid` through the normal ModRepos pipeline.
- If `modid` is omitted, SLP falls back to the first mod in the repository.
- A GitHub repository-root URL is normalized to its usual `refs/heads/modrepo/modrepo.xml` location.
- The detail view reports success or failure clearly. Failures leave the notice open for retrying or ignoring.

### `workshop_mod_install`

```xml
<Primary label="Replace mod" action="workshop_mod_install"
         workshop_id="1234567890" />
```

- Supported on the primary button and intended for migrations.
- Subscribes to and downloads the target item.
- If the trigger contains a valid `workshop_id`, the triggering Workshop item is unsubscribed only after the replacement succeeds.
- Newly discovered Workshop mods are automatically enabled through SLP's normal new-mod behavior.
- The detail view reports success or failure clearly.

### `workshop_mod_subscribe`

```xml
<Secondary label="Install dependency" action="workshop_mod_subscribe"
           workshop_id="1234567890" />
```

- Supported on either the primary or secondary button.
- Subscribes to and downloads the item without uninstalling the mod that triggered the notice.
- Shows clear success or failure feedback in the detail view.
- Combine it with `unless_workshop_id` using the same ID to hide the notice on later launches once the item is installed.

### `open_url`

```xml
<Secondary label="More info" action="open_url" url="https://example.com" />
```

Opens the URL in the user's browser. The notice remains visible.

### `acknowledge` and `dismiss`

```xml
<Primary label="OK" action="acknowledge" />
```

Both actions close the notice for the current session. An empty action is treated the same way. They do not add the notice ID to the permanent ignored list.

## Formatting tags

`long_description` supports:

- `[b]...[/b]`, `[i]...[/i]`, `[u]...[/u]`, and `[strike]...[/strike]`
- `[h1]...[/h1]`, `[h2]...[/h2]`, and `[h3]...[/h3]`
- `[url=https://example.com]link text[/url]`
- `[code]...[/code]`
- `[red]`, `[green]`, `[blue]`, `[yellow]`, and `[cyan]`
- `[list]` with `[*]` list items

`[b]` renders at 1.15× scale because no bold font is shipped. `[i]` currently has no visible effect because no italic font is registered.

XML-special characters in text or attributes must be escaped: use `&amp;`, `&lt;`, `&gt;`, `&quot;`, and `&apos;`. The HTML builder handles this automatically.

## Ignoring and completion

- Every notice has an **Ignore** button. Ignored IDs are stored permanently in `NewsDismissedIds` in the BepInEx config.
- `info` notices can auto-acknowledge after 10 seconds and are not permanently dismissed by that timer.
- Successful install/subscribe actions show a result and a **Close** button. Failures keep the notice available.
- All notice types except `info` block loading until handled.

## Minimal testing feed

```xml
<NewsFeed>
  <Entry id="test-subscribe" type="info" severity="info" heading="Optional Workshop item">
    <short_description>Install an optional item.</short_description>
    <long_description>This notice hides once the item is installed.</long_description>
    <Trigger match_type="always" unless_workshop_id="1234567890" />
    <Actions>
      <Primary label="Subscribe and download" action="workshop_mod_subscribe"
               workshop_id="1234567890" />
      <Secondary label="More info" action="open_url"
                 url="https://steamcommunity.com/sharedfiles/filedetails/?id=1234567890" />
    </Actions>
  </Entry>
</NewsFeed>
```

## Implementation notes

- `NewsRunner.GetActiveNotices(ModList)` fetches and matches notices after the mod list is built and before the autoload countdown.
- The system is skipped when `Platform.IsServer` is true.
- Any number of entries is supported; the UI shows a list when multiple entries are active.
- After a successful repo install, the in-memory mod list is refreshed before configuration continues.
