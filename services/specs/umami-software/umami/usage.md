# Umami Web Analytics

Each enrollment runs a private [Umami](https://umami.is/) with its own PostgreSQL database. Umami is written and maintained by [Umami Software, Inc.](https://github.com/umami-software) and contributors and is released under the MIT license; the provider of this service is its author, who was not asked to take part. UnitySVC Labs lists the official `ghcr.io/umami-software/umami` image (version 3.5.0) and the official `postgres` image (15, Alpine), both pinned by digest, and the platform runs one isolated stack per enrollment. PostgreSQL is released under the PostgreSQL License.

## Set up

1. Enroll in **base**. The enrollment asks for access to its own storage (`service:storage`, read and write). Approve it, or the instance does not start.
2. The form shows a generated **Administrator password**, an **Application secret** and a **Database password**. Keep the generated values or enter your own. Save the administrator password; all three are also shown in the enrollment's instance details. They are application credentials, not your UnitySVC API key.
3. Wait for the instance to become ready. Its address is `https://<enrollment-code>.<apps-domain>` (HTTPS, port 443); the instance details show it. Umami starts after its database, so the first start takes a minute or two. **A new enrollment has a new address and empty data.**
4. Open the address and sign in as `admin` with the administrator password.

Umami normally creates the account `admin` with the published password `umami`. Because the address is public, this instance replaces that password with yours before it starts answering: on a new database it signs in to Umami locally with the published password and changes it, and only then opens to the network. The administrator password is used only then; afterwards the stored password applies, and you change it in Umami under your profile.

## First use

1. In **Websites** choose **Add website**, and enter a name and the domain of your site.
2. Open the website's settings and copy its **tracking code**. It looks like this, with your instance address and the website's ID:

   ```html
   <script defer src="https://<instance>/script.js" data-website-id="<website-id>"></script>
   ```

3. Put it in the `<head>` of every page of your site. Visits appear in Umami within a few seconds.

You do not tell Umami its own address: the script sends its data to the address it was loaded from, and Umami accepts it from any website. Because the address belongs to the enrollment, **after a new enrollment you must put the new address in the tracking code of your sites.**

For the REST API, sign in with `POST /api/auth/login` (`{"username": "admin", "password": "..."}`) and send the returned token as `Authorization: Bearer <token>`. See the [Umami documentation](https://umami.is/docs) for custom events, reports, teams, share links and the API.

## Authentication

Umami checks every request itself: the web interface and the API need your sign-in. Open by design, as any analytics collector is, are the tracking script (`/script.js`), the collection endpoint (`/api/send`), and the pages you publish with a share link. Anyone who knows the instance address can reach the sign-in page, so keep the administrator password strong. UnitySVC adds no other gate. The database is reachable only from inside the instance.

## What is kept, and what is lost

One folder is persistent: a backup of the PostgreSQL database (websites, users, settings and all collected statistics), stored in the enrollment's storage. The database itself runs on temporary disk; the instance writes a backup of it every five minutes and once more when it stops, and restores that backup when it starts. The platform copies the backup to storage about every 30 seconds while the instance runs and once more when it stops. Stopping therefore takes a moment longer while the backup and the copy finish.

- **Umami collects data only while the instance runs.** While it is stopped, your pages still load (the script is loaded with `defer` and fails quietly), but their visits are not recorded.
- A normal stop loses nothing. A crash, or a node failure, can lose the data collected since the last backup, up to five minutes.
- The backup is one file and must stay under 1 GiB, the largest file the platform's storage copy keeps. A busy site collects a lot over time: delete old data in Umami (a website's **Reset** or **Delete** in its settings) or export it before the backup gets that large.
- Removing the enrollment removes its storage. Enrolling again starts with an empty Umami, new passwords and a new address.
- Keep the **Database password** as it is. It is used only between the two containers of your instance.
- Changing the **Application secret** signs everyone out.
- Two-factor authentication is not available in this recipe: Umami needs a separate 64-character hexadecimal encryption key for it, which this recipe does not set.
- The instance sends no email, and Umami does not need any.

## Cost and size

Each of the two containers (Umami and PostgreSQL) runs on the `nano` size (half a core, 512 MiB, 1 GiB of temporary disk) and you pay the platform's cost for both for as long as the instance runs, including idle time. In a local check with those limits Umami used about 100 MiB after starting and 165 to 220 MiB in use, and PostgreSQL 25 to 55 MiB. Labs adds no markup and receives no seller payout.
