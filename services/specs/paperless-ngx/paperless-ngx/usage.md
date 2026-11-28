# Paperless-ngx Document Management with OCR

Each enrollment runs a private [Paperless-ngx](https://docs.paperless-ngx.com/) with its own PostgreSQL database and task broker. Paperless-ngx is written and maintained by the [Paperless-ngx team](https://github.com/paperless-ngx) and contributors and is released under the GPL-3.0 license; the provider of this service is its author, who was not asked to take part. UnitySVC Labs lists the official `ghcr.io/paperless-ngx/paperless-ngx` image (version 3.5.1), the official `postgres` image (17, Alpine) and the official `valkey/valkey` image (9.2.1, Alpine) as the Redis-compatible broker, all pinned by digest, and the platform runs one isolated stack per enrollment. PostgreSQL is released under the PostgreSQL License and Valkey under the BSD 3-Clause license.

## Set up

1. Enroll in **base**. The enrollment asks for access to its own storage (`service:storage`, read and write). Approve it, or the instance does not start.
2. The form shows an **Administrator name** (default `admin`), a generated **Administrator password**, a generated **Secret key** and **Database password**, and the **OCR language** (default `eng`). The administrator values create the account the **first time** Paperless-ngx runs on the data; afterwards the password in the form is ignored and you change it in Paperless-ngx itself (**My Profile**). Save the password; it is also shown in the enrollment's instance details. These are application credentials, not your UnitySVC API key.
3. Wait for the instance to become ready. The first start creates the database and takes a minute or two. Its address is `https://<enrollment-code>.<apps-domain>` (HTTPS, port 443); the instance details show it. **A new enrollment has a new address and an empty archive.**
4. Open the address and sign in with the administrator name and password. Nothing has to be configured with the address: Paperless-ngx accepts the host name it is called with.

## First use

1. Drag a PDF or an image onto the page, or use **Upload documents** on the dashboard. Paperless-ngx reads the text (OCR for scans), creates a searchable PDF/A copy and a thumbnail, and the document appears under **Documents**. A scanned page took about four seconds in a local check; a long scan takes minutes, and the progress is shown on the dashboard.
2. Create **Tags**, **Correspondents** and **Document types** and assign them. Set a tag's matching to **Auto** and Paperless-ngx learns from what you file and suggests it for new documents. The learning runs once an hour.
3. Search the full text from the search box, and save a filtered list as a view.

Documents come in through the web upload, the REST API, or a mail account you add under **Mail** (Paperless-ngx fetches from your IMAP server). There is **no consume folder**: nothing outside the instance can put files into one, so the folder watcher is switched off.

For the REST API create a token under **My Profile** and send it as `Authorization: Token <token>`:

```bash
curl -H "Authorization: Token <token>" https://<instance>/api/documents/
curl -H "Authorization: Token <token>" -F document=@scan.pdf https://<instance>/api/documents/post_document/
```

See the [Paperless-ngx documentation](https://docs.paperless-ngx.com/) for the rest.

## Authentication

Paperless-ngx checks every request itself: the web interface needs your sign-in and the API your token or session; a request without one is answered with the sign-in page or `401`. Sign-up is closed, so only users you create under **Users & Groups** can sign in. Share links you create are open to anyone who has the link, by design. Anyone who knows the instance address can reach the sign-in page, so keep the administrator password strong. UnitySVC adds no other gate. The database and the broker are reachable only from inside the instance.

## What does not work here

- **No email is sent.** The instance has no mail server configured, so "forgot password" mails, emailing a document and the email action of workflows do not work. A signed-in user can still change their password under **My Profile**; an administrator can set another user's password.
- **OCR languages** are the five in the image: English, German, French, Italian and Spanish. Others are not installed.
- **Office documents** (Word, Excel, LibreOffice) and `.eml` files need the separate Tika and Gotenberg servers, which are not part of this stack. PDFs, images and plain text work.
- **The AI features** of Paperless-ngx are off; they need a model provider you would have to configure through settings this recipe does not expose.

## What is kept, and what is lost

Three folders are persistent, in the enrollment's storage: the **documents** (`media`: the originals, the archived PDF/A copies and the thumbnails), a **backup of the PostgreSQL database** (titles, tags, the recognized text, users, settings) and the **trained matching model**. The database itself runs on temporary disk; the instance writes a backup of it every five minutes and once more when it stops, and restores that backup when it starts. The platform copies the three folders to storage about every 30 seconds while the instance runs and once more when it stops. Stopping therefore takes a moment longer while the backup and the copy finish.

- A normal stop loses nothing. A crash, or a node failure, can lose the database changes made since the last backup, up to five minutes. A document uploaded in that window can then be on disk without its database entry; upload it again.
- **The search index is not kept; it is rebuilt at every start** from the text in the database, before the web interface answers. This is on purpose: a stored index would grow without limit (the storage copy never deletes the files the index replaces) and a half-copied one would come back empty. Rebuilding 5,000 documents of about two pages each added about ten seconds to the start in a local check. The platform gives a start about two minutes, so a very large archive (tens of thousands of documents) may fail to start; contact UnitySVC support before you get there.
- The task history, the logs and Paperless-ngx's own export folder are temporary and start empty.
- The backup is one file and must stay under 1 GiB, the largest file the platform's storage copy keeps. The recognized text of every document is in it; 5,000 two-page documents made a backup of about 12 MiB.
- Files are copied, never deleted from storage. **A document you delete in Paperless-ngx disappears from the archive, but its files stay in the enrollment's storage** until the enrollment is removed, and they are put back on the instance's disk at the next start, where Paperless-ngx ignores them (its sanity check reports them as orphaned files). If you rename documents through a storage path, the files under the old names stay in storage as well.
- Empty files, symbolic links and files over 1 GiB are not stored. Paperless-ngx keeps nothing it needs in such files (its empty lock files are created again); a single document over 1 GiB would not survive a stop.
- While it runs, the instance holds the documents on its temporary disk (5 GiB on this size), together with the files being processed. Keep the archive well below that.
- Removing the enrollment removes its storage. Enrolling again starts with an empty Paperless-ngx and new passwords. Download your documents first (select all, **Download**).

## Cost and size

The Paperless-ngx container runs on the `small` size (one core, 2 GiB of memory, 5 GiB of temporary disk); PostgreSQL and the broker each run on `nano` (half a core, 512 MiB, 1 GiB of temporary disk). You pay the platform's cost for all three for as long as the instance runs, including idle time. In a local check with those limits Paperless-ngx used about 480 MiB idle and about 700 MiB during OCR of an eight-page scan, PostgreSQL 30 to 60 MiB and the broker 6 to 12 MiB. The hourly training of the matching model is the heaviest task: with 5,000 documents it took 80 seconds and brought the container to about 1.5 GiB. With a much larger archive the training can run out of memory; uploads, OCR and search keep working, but automatic matching stops improving. Labs adds no markup and receives no seller payout.
