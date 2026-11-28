# Wakapi Coding Time Statistics

Each enrollment runs a private [Wakapi](https://wakapi.dev) server. Wakapi is written and maintained by [Ferdinand Mütsch](https://github.com/muety) and contributors and is released under the MIT license; the provider of this service is its author. UnitySVC Labs lists the official `ghcr.io/muety/wakapi` image (version 2.18.2, pinned by digest), and the platform runs one isolated instance per enrollment. The author was not asked to take part in this listing, and nothing here means the author endorses it or supports it.

Wakapi is a backend for coding-time statistics that speaks the WakaTime protocol. The WakaTime plugin of your editor sends it a "heartbeat" while you work, and Wakapi shows the time you spent per project, language, editor, machine and operating system.

Your accounts and statistics are kept in the `wakapi` folder of your **home**, not in the enrollment. They survive stopping the instance, and also removing it and enrolling again: a new enrollment starts a new instance at a new address, and it loads the same data.

**Read "The first account" before you start the instance for the first time.** Sign-up is open until you close it.

## Set up

1. Enroll in **base**. The enrollment asks for access to the `wakapi/` folder of your home (`service:home`, read and write). Approve it, or the instance does not start. You can run one Wakapi instance at a time.
2. The form has three settings. **Password salt** is generated; keep it or enter your own, and save it now: the platform keeps it as a secret and does not show it again. It is not a sign-in password and not your UnitySVC API key (see **The password salt** below). **Allow sign-up** starts as `true`. **Public address** is optional; leave the placeholder.
3. Wait for the instance to become ready; it starts in a few seconds. Its address is `https://<enrollment-code>.<apps-domain>` (HTTPS, port 443); the instance details show it. **A new enrollment has a new address.**
4. Open the address at once and create your account, as described next.

## The first account

Wakapi has no setting for an administrator's name or password, so this service has no such parameter. Accounts are created on the **Sign up** page, and **the first account that signs up is the administrator**.

1. Open the address, choose **Sign up**, and enter a user name and a password (at least 6 characters). The email field is optional; leave it empty, since this instance sends no mail. If your data is already in your home from an earlier enrollment, skip this step and sign in with the account you created then.
2. **Close sign-up.** Edit the enrollment, set **Allow sign-up** to `false`, and save. The instance restarts and keeps your data; you have to sign in again.
3. Sign in.

**Until you close sign-up, anyone who reaches the instance address can sign up too.** The address is not listed anywhere, but it is not a secret either. What that exposes, as checked locally:

- Someone who signs up **before you** on a new instance becomes the administrator. The administrator has no management pages in Wakapi, but its API key may read other users' statistics. If you are not sure you were first, cancel the enrollment, delete the `wakapi/` folder in your home and start again.
- Someone who signs up **after you** gets an ordinary account: it cannot read your statistics (the API answered 401 and 403), but it uses your instance and the storage in your home. Wakapi accepts at most five sign-up attempts an hour.
- With **Allow sign-up** set to `false`, a sign-up is refused ("Registration is disabled on this server") and existing accounts still sign in. Accounts created before you closed it remain; Wakapi has no page to list or remove other accounts.
- A new enrollment starts with **Allow sign-up** at `true` again, also when your data is already there. Set it to `false` in the enrollment form, or right after the instance starts.

To let another person in later, generate an invite link in **Settings > Account** (**Invite Friends**); it works once, for 24 hours, also while sign-up is closed. The link starts with the **Public address** setting, so either set that to your instance address or replace `https://instance.invalid` in the link by hand.

## Connect your editor

1. Install the [WakaTime plugin](https://wakatime.com/plugins) for your editor or IDE. You do not need a WakaTime account.
2. Copy your API key: the **API Key** button at the top of the dashboard shows it, and **Settings > API Keys** lets you reset it or add more.
3. Put the address of your instance, followed by `/api`, and the key in `~/.wakatime.cfg` (on Windows, `.wakatime.cfg` in your user folder):

   ```ini
   [settings]
   api_url = https://<instance>/api
   api_key = <your Wakapi API key>
   ```

4. Work in the editor for a minute, then open the dashboard (`https://<instance>/summary`). After a new enrollment, change `api_url` to the new address; the key stays the same.

To report to WakaTime and to Wakapi at the same time, keep your WakaTime key in `[settings]` and add Wakapi as a second destination, as the Wakapi README describes:

```ini
[settings]
api_key = <your WakaTime API key>

[api_urls]
.* = https://<instance>/api|<your Wakapi API key>
```

## Use it

- **Dashboard** at `/summary`: time per project, language, editor, machine, operating system and label, for a period you choose.
- **REST API.** Send the API key base64-encoded in a `Basic` authorization header, or plain as an `api_key` query parameter:

  ```bash
  curl -H "Authorization: Basic $(printf %s '<your Wakapi API key>' | base64)" \
       "https://<instance>/api/summary?interval=today"
  ```

  The plugins post heartbeats to `/api/heartbeat` and `/api/users/current/heartbeats.bulk`. Part of the WakaTime API is available under `/api/compat/wakatime/v1/`, and the interactive reference is at `https://<instance>/swagger-ui`.
- **Settings**: aliases and labels for projects, custom language mappings, badges and sharing, an import of your existing WakaTime data, and relaying heartbeats to WakaTime.

The [Wakapi README](https://github.com/muety/wakapi#readme) describes the rest.

## Authentication

Wakapi checks every request itself: the dashboard and settings need your sign-in, and the API needs an API key; without one it answers 401. A heartbeat with a wrong key is refused. UnitySVC adds no other gate, so anyone who knows the instance address can reach the start page and the sign-in page. Use a strong password.

- The start page and `/leaderboard` are public pages. The leaderboard lists only users who opt in under **Settings > Permissions**; nobody is listed by default.
- **No mail.** This recipe configures no mail server. "Forgot password" answers "Mailing is disabled on this server", and there are no weekly report mails. You can change your password in **Settings > Account** while you know the current one. If you forget it, there is no way to recover the account.
- Restarting the instance signs everyone out of the web interface (Wakapi makes new cookie keys at each start). API keys are not affected, so editors keep reporting.

## The password salt

Wakapi mixes the salt into every stored password hash. **If the salt changes, no existing password is accepted any more**; checked locally, the same sign-in answered 401 after a change and worked again when the old value was put back. API keys and the statistics are not affected, so your editors keep reporting, but you cannot sign in to see the result, and without mail there is no reset.

Because your data outlasts the enrollment and a new enrollment generates a new salt, **enter your saved salt when you enroll again**. The platform does not show it to you later, so save it when you first enroll.

## What is kept, and what is lost

Wakapi's data folder is persistent. It holds one file, the SQLite database `wakapi.db`, with the accounts, API keys, settings and heartbeats. It is stored in `wakapi/` in your home, where you can see it in your files while the instance is stopped. The platform restores it before Wakapi starts, copies it back about every 30 seconds while it changed, and makes a last copy when the instance stops. Stopping therefore takes a moment longer while the copy finishes, and the instance shows as `draining`.

- **One instance at a time.** The folder belongs to your account, not to one enrollment, and two running instances would overwrite each other's copy. The service allows one active enrollment per account.
- A crash, or a node failure, can lose up to the last 30 seconds of heartbeats. A normal stop loses nothing.
- The database is one self-contained file (in this image SQLite runs in rollback-journal mode, not with a write-ahead log), so the copy in your home is complete. SQLite's temporary files (`*-wal`, `*-shm`, `*-journal`) are not stored.
- The database must stay under 1 GiB: a larger file is not stored. In a local check 20,000 heartbeats took about 8 MB. Wakapi keeps heartbeats forever by default; **Clear Data** under **Settings > Danger Zone** removes all of your heartbeats.
- Do not edit the file in `wakapi/` while the instance is stopped unless you mean to: the next start loads what is there.
- The three enrollment settings are read at every start, so changing one takes effect after the restart. Your account, its password and the API keys live in the database, not in the settings.
- Accounts are never removed for inactivity (the image would otherwise remove an account after 12 months without activity).
- Nothing is deleted from your home automatically, including when you cancel the enrollment. Delete the `wakapi/` folder yourself when you no longer want the data.

## Cost and size

The instance runs on the `nano` size (half a core, 512 MiB, 1 GiB of temporary disk) and you pay the platform's cost for it for as long as it runs, including idle time. In a local check Wakapi used about 25 MiB when just started and 60 to 225 MiB after sign-ins and while receiving 20,000 heartbeats (hashing a password takes memory for a moment). Labs adds no markup and receives no seller payout. The data in your home is not billed while that feature is in preview.
