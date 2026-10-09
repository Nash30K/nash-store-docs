# Database

## Nothing to import

The schema builds itself. The resource runs its schema pass at every start and creates the 31
tables it needs. On success it prints:

```
[nash_phone] schéma DB prêt (31 tables).
```

If it also had catching up to do, the same line reports it:

```
[nash_phone] schéma DB prêt (31 tables, 4 colonne(s) ajoutée(s), 2 index ajouté(s)).
```

A column that was widened (see `TYPES` below) is not counted on that line.

`sql/nash_phone.sql` contains the same `CREATE TABLE` statements. **It is not read by the
resource.** It exists as a reference, and so you can prepare the database by hand before the
first boot if your workflow requires it. It is not a required step, and importing it does not
replace a first start (see [The `.sql` file is not enough](#the-sql-file-is-not-enough)).

The MySQL account used by `oxmysql` must be able to run `CREATE TABLE`, `ALTER TABLE` and to
read `information_schema`. Without `ALTER`, new columns and indexes are never added and the
apps that need them answer with errors.

## Tables

All 31 tables are prefixed `nash_phone_`. Every per-character table is keyed on `owner`, the
identifier returned by your framework (ESX license, QB citizenid).

### Device and player

| Table | Purpose |
|---|---|
| `nash_phone_phones` | One row per character: `owner` → `number`. The number is `UNIQUE` |
| `nash_phone_settings` | All per-player settings, one JSON blob per character |
| `nash_phone_home` | Home screen: page layout, dock, installed applications |
| `nash_phone_setup` | First-open setup wizard: `step` is stored from the first screen, so a player who disconnects mid-way resumes where they left off |
| `nash_phone_faceid` | Which character's face each phone recognises: `device` (the item's serial number) → `owner`. One face per phone, the first character to use it keeps it. See [Settings > Face ID](../apps/README.md#system) |
| `nash_phone_screentime` | Real screen time, one row per character and per day |

### Communication

| Table | Purpose |
|---|---|
| `nash_phone_contacts` | Contacts. Unique on `(owner, number)` |
| `nash_phone_messages` | Text messages, addressed by **phone number**. `del_sender` / `del_receiver` let each side delete its own copy of a thread |
| `nash_phone_calls` | Call log. `app_id` says which app placed the call, so ChatApp does not list dialler calls |
| `nash_phone_mail_accounts` | One frozen e-mail address per character, generated from the RP name |
| `nash_phone_mails` | One row **per mailbox**: sending writes two rows, the recipient's inbox and the sender's sent folder |

### Media and content

| Table | Purpose |
|---|---|
| `nash_phone_gallery` | Photos and videos. Stores the hosted **URL**, never the file. `kind` is `photo` or `video`; `album` says where the media came from (see below) |
| `nash_phone_notifications` | Phone notifications, purged on a schedule (`Config.Notifications`) |
| `nash_phone_appdata` | Notes, Calendar and Reminders content, plus extra data of the Contacts app: one JSON blob per character and per app, 256 KB at most |
| `nash_phone_bank` | Wallet ledger shown in the phone |

The `album` column of `nash_phone_gallery` records where a media came from:

| `album` | Written by |
|---|---|
| `Camera` | The Camera, the Snapz save button, custom apps and the `SaveToGallery` export. It is also the column's default |
| `Import` | A photo a player added from a link in Photos. Its address is always on a host listed in `Config.Gallery.Import.AllowedHosts` (Fivemanage by default), never a Discord link |
| `AirDrop` | A photo or video received from a nearby player and accepted |

The **Recently Saved** collection of Photos lists the `Import` and `AirDrop` rows. **Duplicate**
in Photos writes a new row with the same `url`, `thumb`, `album` and `created_at` as the
original: two rows, one hosted file. Deleting a row never deletes the file from the host. No new
column was needed: `album` has existed since the first release, as `VARCHAR(64)`.

### byCloud account

| Table | Purpose |
|---|---|
| `nash_phone_icloud` | Accounts: e-mail, salted password hash, identity fields, and `owner` (the character who created it) |
| `nash_phone_icloud_backup` | Keychain backup attached to an account |

### Social networks

The social tables are generic and carry an `app_id` column: the applications share one model
and differ only in presentation.

| Table | Purpose |
|---|---|
| `nash_phone_social_accounts` | One account per app and per identifier. Unique on `(app_id, identifier)` |
| `nash_phone_social_messages` | Private messages, including snaps (`ephemeral`, `opened_at`) |
| `nash_phone_social_posts` | Posts, with denormalised `like_count` / `comment_count` |
| `nash_phone_social_likes` | Primary key `(post_id, liker)` |
| `nash_phone_social_comments` | Comments |
| `nash_phone_social_follows` | Primary key `(app_id, follower, target)` |
| `nash_phone_social_notifs` | Social notifications |
| `nash_phone_social_stories` | Stories. No expiry column: it is computed from `created_at` |
| `nash_phone_social_story_views` | Who saw which story |

### Services

| Table | Purpose |
|---|---|
| `nash_phone_service_requests` | Requests sent to jobs from the Services app. Live state (who is on duty) stays in server memory and is not persisted |

### Cindr (dating app)

| Table | Purpose |
|---|---|
| `nash_phone_dating_profiles` | One dating profile per account: display name, age, bio, photos, visibility, age range sought, optional neighbourhood |
| `nash_phone_dating_swipes` | Who liked or passed on whom. Primary key `(app_id, swiper, target)` |
| `nash_phone_dating_matches` | Mutual likes |
| `nash_phone_dating_reports` | Reports sent from the app, with their reason (see `Config.Social.Dating.reports`) |

## Migrations

This is the part that matters on a server that is already running.

{% hint style="warning" %}
`CREATE TABLE IF NOT EXISTS` does **nothing** to a table that already exists. A column added
in a later version would never reach an installation that has been live for months. That is
why the resource keeps four separate lists, not one.
{% endhint %}

| List | What it does | When it acts |
|---|---|---|
| `DDL` | The 31 `CREATE TABLE IF NOT EXISTS` statements | Creates missing tables. Skips existing ones entirely |
| `COLUMNS` | 22 `{ table, column, definition }` entries | Adds any column that is missing, on fresh **and** existing databases |
| `INDEXES` | 9 `{ table, index, columns }` entries | Adds any index that is missing |
| `TYPES` | 2 `{ table, column, expected type, definition }` entries | **Widens** a column that exists but is too narrow. Never narrows one |

Every catch-up pass first asks `information_schema` what already exists (one query for the
columns, one for the indexes, one for the column types), then runs only the `ALTER TABLE`
statements that are actually needed. On a database that is already up to date, a boot costs
the `CREATE TABLE IF NOT EXISTS` statements (which change nothing) and three
`information_schema` queries, and no `ALTER TABLE` at all.

{% hint style="info" %}
`COLUMNS` only **adds** a column that is absent. It never looks at the type of a column that
is already there, which is what `TYPES` is for. The distinction matters: `media_url` shipped
as `VARCHAR(512)` while the server accepts URLs up to 1024 characters, so a long address made
the INSERT fail in strict mode and the post was refused without a word. `TYPES` widens it to
`TEXT` on databases that already have the narrow column.
{% endhint %}

### Columns added after first release

| Table | Column | What it carries |
|---|---|---|
| `nash_phone_gallery` | `kind`, `duration`, `thumb` | Camera videos, their length and thumbnails |
| `nash_phone_contacts` | `note`, `created_at` | Free note on a contact card, date added by the `AddContact` export |
| `nash_phone_messages` | `del_sender`, `del_receiver` | Per-side thread deletion |
| `nash_phone_calls` | `app_id` | Per-app call log. Existing rows default to `phone` |
| `nash_phone_social_accounts` | `display_name`, `avatar_url`, `bio`, `private`, `map_share` | Social profile, private account, and map sharing (`0` never answered, `1` ghost, `2` friends) |
| `nash_phone_social_messages` | `ephemeral`, `opened_at`, `mtype` | Snaps and shared posts |
| `nash_phone_icloud` | `first_name`, `last_name`, `age`, `owner` | Identity asked by the setup wizard, and the character who created the account |
| `nash_phone_setup` | `device` | Serial number of the phone the setup was done on, so a new phone asks for it again |
| `nash_phone_phones` | `announced` | Whether `nash-phone:phoneNumberGenerated` has fired for that number. Existing rows default to `1`, so no event fires for them after the update |

### Indexes added after first release

An index that is missing breaks nothing: it makes things slower, and slower in proportion to
how long your server has been running.

| Table | Index | Query it serves |
|---|---|---|
| `nash_phone_messages` | `pair_at` | Opening a conversation and marking it read |
| `nash_phone_messages` | `created_at` | Joining the last message of every thread in the conversation list. Without it, opening Messages re-reads the whole table, all players included |
| `nash_phone_messages` | `recv_read` | The unread-SMS counter, which runs on every phone open |
| `nash_phone_gallery` | `owner_at` | Camera roll pagination |
| `nash_phone_bank` | `owner_at` | Wallet history pagination |
| `nash_phone_calls` | `owner_at` | Recents in the Phone app |
| `nash_phone_notifications` | `owner_read_at` | Unread notifications on boot and in the Notifications app |
| `nash_phone_social_messages` | `app_recv_at` | The "I am the recipient" half of the social conversation list |
| `nash_phone_social_notifs` | `post` | Cascading deletes when a post is removed |

{% hint style="warning" %}
These indexes are created with `ALTER TABLE` on the **first boot after the update**. On an
installation with millions of rows, that single start will be noticeably longer. It happens
once. An index that cannot be created is reported in the console and the boot continues.
{% endhint %}

### The `.sql` file is not enough

`sql/nash_phone.sql` is kept in sync with the full schema, tables, columns, types and indexes
alike. It is still only a starting point: the resource runs its catch-up passes at **every**
boot, and they are what keeps a live database correct across updates.

**Importing the SQL file never replaces starting the resource.** A version released after your
import will add its own columns and indexes on first boot, and the file you imported knows
nothing about them.

## Verifying the schema

The boot line is the first check. `schéma DB prêt (31 tables)` means the schema pass ran to
the end, and an index that could not be added gets its own line (`index … sur … non créé : …`).
If the `schéma DB prêt` line never appears, the pass stopped on a database error: read the
error printed above it, often a MySQL account without the `CREATE` or `ALTER` right.

To look further, query the database directly:

```sql
-- 31 rows expected
SHOW TABLES LIKE 'nash\_phone\_%';

-- the columns of one table, to compare with "Columns added after first release"
SHOW COLUMNS FROM nash_phone_gallery;

-- the indexes of one table, to compare with "Indexes added after first release"
SHOW INDEX FROM nash_phone_messages;
```

To spot a table growing abnormally on a long-running server, count its rows, all characters
included:

```sql
SELECT COUNT(*) FROM nash_phone_notifications;
```

{% hint style="info" %}
All 31 tables listed above are created by the resource and in active use, including
`nash_phone_setup`, `nash_phone_service_requests`, `nash_phone_faceid` and the
`nash_phone_dating_` tables. **Do not drop them.**
{% endhint %}

## Sizing and limits

- **Phone numbers** are stored in `VARCHAR(16)`. The generated form is
  `<prefix>-<digits>`, so `Config.Phone.prefix = '555'` with `digits = 7` gives `555-1234567`,
  11 characters. If your own values push the total past 16, number assignment either fails or
  the number is truncated depending on the SQL mode. Nothing checks this at boot, so keep
  the prefix, the dash and `digits` at 16 characters or fewer in total.
- **Media URLs** live in `TEXT` columns. The phone stores links, never the files themselves,
  so the database stays small regardless of how many photos players take.
- **Notifications** are purged by a task that starts a minute after boot and then runs
  hourly, driven by `Config.Notifications.keepReadDays` (3) and `keepDays` (30) in
  `config/main.lua`. Set either to `0` to disable that half of the purge. Without a purge, the
  table grows forever. On top of that, `config/notifications.lua` deletes a player's
  notifications when they disconnect (`purgeOnDrop`) and when they connect (`purgeOnJoin`),
  both `true` by default.
- **Social notifications** (`nash_phone_social_notifs`) have their own purge, set in
  `Config.Social.Notifs` (`config/social.lua`).

## Upgrading

1. Back up the database.
2. Replace the resource folder, keeping your own files: `config/`, `server/bridge/custom.lua`
   if you filled it, any language file you added to `locales/` (your edits to `fr.json` and
   `en.json` are replaced, note them first), and your PNG files in `service_icons/`.
3. Restart the resource, and read the `schéma DB prêt` line: it tells you exactly how many
   columns and indexes were added.
4. Check that no `index … non créé` line appeared. To confirm further, see
   [Verifying the schema](#verifying-the-schema).

{% hint style="danger" %}
Never change `Config.Framework` on a live server. Every table is keyed on the identifier the
framework hands out; switching framework leaves every existing row unreachable. See
[Requirements](requirements.md#framework-and-inventory-selection).
{% endhint %}
