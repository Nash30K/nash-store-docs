# Database

## Nothing to import

The schema builds itself. `server/db/schema.lua` runs at every start of the resource and
creates the 26 tables it needs. On success it prints:

```
[nash_phone] schéma DB prêt (26 tables).
```

If it also had catching up to do, the same line reports it:

```
[nash_phone] schéma DB prêt (26 tables, 4 colonne(s) ajoutée(s), 2 index ajouté(s)).
```

`sql/nash_phone.sql` contains the same `CREATE TABLE` statements. **It is not read by the
resource.** It exists as a reference, and so you can prepare the database by hand before the
first boot if your workflow requires it. It is not a required step, and importing it does not
replace a first start (see [The `.sql` file is not enough](#the-sql-file-is-not-enough)).

The MySQL account used by `oxmysql` must be able to run `CREATE TABLE`, `ALTER TABLE` and to
read `information_schema`. Without `ALTER`, new columns silently fail to appear and the apps
that need them answer with errors.

## Tables

All 26 tables are prefixed `nash_phone_`. Every per-character table is keyed on `owner`, the
identifier returned by your framework (ESX license, QB citizenid).

### Device and player

| Table | Purpose |
|---|---|
| `nash_phone_phones` | One row per character: `owner` → `number`. The number is `UNIQUE` |
| `nash_phone_settings` | All per-player settings, one JSON blob per character |
| `nash_phone_home` | Home screen: page layout, dock, installed applications |
| `nash_phone_setup` | First-open setup wizard: `step` is stored from the first screen, so a player who disconnects mid-way resumes where they left off |
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
| `nash_phone_gallery` | Photos and videos. Stores the hosted **URL**, never the file. `kind` is `photo` or `video` |
| `nash_phone_notifications` | Phone notifications, purged on a schedule (`Config.Notifications`) |
| `nash_phone_appdata` | Notes, Calendar, Reminders and Mail content: one JSON blob per character and per app |
| `nash_phone_bank` | Wallet ledger shown in the phone |

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

## Migrations

This is the part that matters on a server that is already running.

{% hint style="warning" %}
`CREATE TABLE IF NOT EXISTS` does **nothing** to a table that already exists. A column added
in a later version would never reach an installation that has been live for months. That is
why `server/db/schema.lua` holds four separate lists, not one.
{% endhint %}

| List | What it does | When it acts |
|---|---|---|
| `DDL` | The 26 `CREATE TABLE IF NOT EXISTS` statements | Creates missing tables. Skips existing ones entirely |
| `COLUMNS` | 20 `{ table, column, definition }` entries | Adds any column that is missing, on fresh **and** existing databases |
| `INDEXES` | 9 `{ table, index, columns }` entries | Adds any index that is missing |
| `TYPES` | 2 `{ table, column, expected type, definition }` entries | **Widens** a column that exists but is too narrow. Never narrows one |

Every catch-up pass first asks `information_schema` what already exists (one query for the
columns, one for the indexes, one for the column types), then runs only the `ALTER TABLE`
statements that are actually needed. On a database that is already up to date, a boot costs
three `information_schema` queries and nothing else.

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

With `Config.Debug = true`, `/phoneschema` compares the live database against the expected
lists and reports missing tables, missing columns and missing indexes, with what each one
costs you.

```
/phoneschema
```

{% hint style="info" %}
`/phoneschema` currently checks 24 of the 26 tables. It reports `nash_phone_setup` and
`nash_phone_service_requests` as `en trop` ("unknown table, probably from an older
version"). They are neither unknown nor old: both are created by `server/db/schema.lua` and
are in active use. **Do not drop them.**
{% endhint %}

`/phonebase` prints a real `COUNT(*)` per table, all characters included: useful to spot a
table growing abnormally on a long-running server.

## Sizing and limits

- **Phone numbers** are stored in `VARCHAR(16)`. The generated form is
  `<prefix>-<digits>`, so `Config.Phone.prefix = '555'` with `digits = 7` gives `555-1234567`,
  11 characters. If your own values push the total past 16, number assignment either fails or
  the number is truncated depending on the SQL mode. `/phonecheck` refuses to go green on
  that case.
- **Media URLs** live in `TEXT` columns. The phone stores links, never the files themselves,
  so the database stays small regardless of how many photos players take.
- **Notifications** are purged by a task that starts a minute after boot and then runs
  hourly, driven by
  `Config.Notifications.keepReadDays` (3) and `keepDays` (30). Set either to `0` to disable
  that half of the purge. Without a purge, the table grows forever.

## Upgrading

1. Back up the database.
2. Replace the resource folder, keeping your `config/` files.
3. Restart the resource, and read the `schéma DB prêt` line: it tells you exactly how many
   columns and indexes were added.
4. Run `/phoneschema` with `Config.Debug = true` to confirm, then set `Config.Debug` back to
   `false`.

{% hint style="danger" %}
Never change `Config.Framework` on a live server. Every table is keyed on the identifier the
framework hands out; switching framework leaves every existing row unreachable. See
[Requirements](requirements.md#framework-and-inventory-selection).
{% endhint %}
