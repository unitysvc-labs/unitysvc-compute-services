# Grocy Groceries and Household Management

Each enrollment runs a private [Grocy](https://grocy.info/) server. Grocy is written and maintained by [Bernd Bestel](https://berrnd.de) and is released under the MIT license; the provider of this service is its author, who was not asked to take part. Grocy publishes no container image of its own: UnitySVC Labs lists the community image its README points to, `lscr.io/linuxserver/grocy` by LinuxServer.io (Grocy 4.7.1, pinned by digest), and the platform runs one isolated instance per enrollment.

Your household data is kept in the `grocy` folder of your **home**, not in the enrollment. It survives stopping the instance, and also removing it and enrolling again: a new enrollment starts a new instance at a new address, and it loads the same data.

## Set up

1. Enroll in **base**. The enrollment asks for access to the `grocy/` folder of your home (`service:home`, read and write). Approve it, or the instance does not start. You can run one Grocy instance at a time.
2. The form shows a generated **Administrator password**. It becomes the password of Grocy's administrator, `admin`, the **first time** Grocy runs on your data. If the database already exists, because you enrolled before or restarted, the parameter is ignored and the existing password still applies; change it in Grocy under the user menu (top right) > **Change password**. Save the password now: the platform keeps it as a secret and does not show it again. It is an application credential, not your UnitySVC API key.
3. Set **Currency** to the three-letter code of the currency your prices are in (default `USD`). Grocy only uses it to show amounts. You can change it later; that restarts the instance and keeps your data.
4. Wait for the instance to become ready; a start takes well under a minute. Its address is `https://<enrollment-code>.<apps-domain>` (HTTPS, port 443); the instance details show it. **A new enrollment has a new address.** Grocy needs no setting for it.
5. Open the address and sign in as `admin` with the administrator password.

## Use it

1. Under **Manage master data**, add your **Locations** (pantry, fridge, freezer), check the **Quantity units**, and add **Products**, each with a location and a unit. With a barcode scanner, or the camera button next to a barcode field, Grocy can look an unknown barcode up on Open Food Facts and fill in the product for you.
2. Record what you buy with **Purchase** and what you use with **Consume**. The **Stock overview** then shows what you have, what is due soon, what is overdue and what is below its minimum amount.
3. Use the **Shopping list** for what to buy, add **Recipes** and a **Meal plan**, and track **Chores**, **Tasks**, **Batteries** and **Equipment**. Parts you do not need can be hidden with Grocy's feature settings.
4. To share the instance with your household, create their accounts under the settings menu (the wrench at the top right) > **Manage users**. Grocy has no self-registration, so only accounts you create can sign in.

Grocy has a REST API under `https://<instance>/api/`, with a browser for it at `https://<instance>/api`. Create a key under the settings menu > **Manage API keys** and send it in the `GROCY-API-KEY` header:

```bash
curl -H "GROCY-API-KEY: <api-key>" https://<instance>/api/stock
curl -H "GROCY-API-KEY: <api-key>" https://<instance>/api/system/info
```

Mobile and other third-party clients listed on the [Grocy website](https://grocy.info/addons) take the instance address and an API key; after a new enrollment, give them the new address.

## Authentication

Grocy checks every request itself: the web interface needs your sign-in, and the API needs an API key or a signed-in session; without one it answers `401`. Anyone who knows the instance address can reach the sign-in page, and Grocy does not slow down or lock out repeated sign-in attempts, so keep the administrator password long and random. Upstream describes Grocy as a household application, not one hardened for public exposure, and says it should never be reachable from the internet without authentication; this recipe keeps Grocy's authentication on and has no setting to turn it off. UnitySVC adds no other gate.

A new Grocy creates the `admin` account with the published password `admin` and has no environment setting or command to change it. Grocy does, however, still create the account from the user name and password settings of its older versions when they are present. This recipe adds those two settings to Grocy's `config.php` before a new database is created, and Grocy reads your **Administrator password** for them from the instance's environment, so the account is created with your password and the default one is never usable. The instance does not start if the password is missing. Once the database exists the settings have no effect.

## What is kept, and what is lost

Grocy's data folder is persistent: the SQLite database (`grocy.db`, which holds the products, stock, recipes, chores, users, API keys and sessions), the settings file (`config.php`) and the uploaded files such as product pictures, recipe pictures and equipment manuals (`storage/`). They are stored in `grocy/` in your home, where you can see them in your files while the instance is stopped. The platform restores them before Grocy starts, copies new and changed files back about every 30 seconds while it runs, and makes a last copy when the instance stops. Stopping therefore takes a moment longer while the copy finishes, and the instance shows as `draining`.

- **One instance at a time.** The folder belongs to your account, not to one enrollment, and two running instances would overwrite each other's copy. The service allows one active enrollment per account.
- A crash, or a node failure, can lose up to the last 30 seconds of changes. A normal stop loses nothing.
- Files are copied, never deleted from your home. The file of a picture or manual you remove in Grocy can stay in `grocy/storage/`; Grocy no longer shows it.
- SQLite's temporary files (`*-journal`, `*-wal`, `*-shm`) and Grocy's page cache (`viewcache/`) are not stored. Grocy rebuilds them, and leaving them out keeps the stored database consistent. Grocy writes every change into `grocy.db` itself, so the stored file is complete.
- Do not edit the files in `grocy/` while the instance is stopped: the next start restores them over what you changed.
- Empty files, symbolic links and files over 1 GiB are not stored. Grocy does not need any of them: an upload is at most 100 MB.
- Nothing is deleted from your home automatically, including when you cancel the enrollment. Delete the `grocy/` folder yourself when you no longer want the data.
- **Only the currency is a parameter.** Grocy's other settings (default language, feature switches, first day of the week) live in `config.php` and keep their defaults; each user can choose a language and many preferences in Grocy's own settings pages.
- **No email.** Grocy sends no email and needs none.
- Barcode lookup asks the public Open Food Facts service from the instance, so it needs the instance's outbound internet access.

## Cost and size

The instance runs on the `nano` size (half a core, 512 MiB, 1 GiB of temporary disk) and you pay the platform's cost for it for as long as it runs, including idle time. In a local check limited to that size Grocy used about 40 MiB idle and at most about 105 MiB while serving many pages at once, and a first start took under 10 seconds. Labs adds no markup and receives no seller payout. The data in your home is not billed while that feature is in preview.
