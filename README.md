# App Password Generator

A small WordPress plugin that creates, tests and revokes **Application Passwords** from one admin screen.

It was built for a common problem: you click **Add New Application Password** on your profile page, the password gets created, but the key **never appears on screen**. This plugin creates the password on the server and shows it in a normal page load, so it does not depend on the JavaScript or REST API request that usually breaks.

- Shows the new password with one-click **Copy** buttons (username, password, site URL)
- **Tests the password automatically** by logging in to your site's REST API, and explains what's wrong if it fails
- Lets administrators create passwords for **other users** (useful for a dedicated, lower-permission "bot" user)
- Lists existing passwords with **last used** date and IP, plus **Revoke** and **Revoke all**
- **Environment check** for the usual blockers (HTTPS, disabled by a plugin, etc.)
- Stores nothing and adds nothing to your public site. Deleting the plugin leaves no trace.

---

## Requirements

| | Minimum |
|---|---|
| WordPress | 5.6 (tested up to 7.1) |
| PHP | 7.0 (tested on 8.3) |
| Site | HTTPS, or `WP_ENVIRONMENT_TYPE` set to `local` |
| Access | Administrator (`manage_options`) |

---

## Installation

### Option A: Upload the zip (easiest)

1. On GitHub, click **Code → Download ZIP** (or download a zip from **Releases** if you created one).
2. In WordPress go to **Plugins → Add New → Upload Plugin**.
3. Choose the zip, click **Install Now**, then **Activate**.

> The folder inside a GitHub zip is named like `app-password-generator-main`. That is fine; WordPress installs it normally.

### Option B: Manual upload (File Manager / FTP)

1. Create the folder `wp-content/plugins/app-password-generator/`.
2. Upload `app-password-generator.php` into it. The other files are optional.
3. Activate it under **Plugins**.

### Option C: Git (developers)

```bash
cd wp-content/plugins
git clone https://github.com/YOUR-USERNAME/app-password-generator.git
```

Then activate it under **Plugins**.

---

## How to use

### Create a password

1. Go to **Tools → App Password Generator**. There is also an **Open** link on the Plugins page.
2. *(Optional)* Use **Managing passwords for** to pick another user, then click **Switch user**.
3. Type a **Name** that tells you where the key is used, for example `Claude Bridge` or `Zapier`.
4. Leave **Connection test** ticked.
5. Click **Generate Application Password**.
6. **Copy the password immediately.** WordPress stores only a hashed copy, so it can never be shown again.

### Use it in your app or plugin

Most apps ask for three things:

| Field | What to enter |
|---|---|
| Site URL | Your site address, e.g. `https://example.com` |
| Username | Your WordPress **username** (not your email) |
| Password | The application password. Spaces are optional; WordPress ignores them. |

Do **not** use your normal login password. Application passwords only work for API access and can't be used to log in to wp-admin.

### Understanding the connection test

| Result | What it means | What to do |
|---|---|---|
| ✅ **Test passed** | The password works with the REST API. | Nothing. Use it. |
| ⚠️ **Could not connect to itself** | Your host blocks the site from calling itself. The password is probably fine. | Test from your app, or with the cURL command shown on screen. |
| ❌ **Login details never reached WordPress** | The server is stripping the `Authorization` header. | See [Authorization header is being stripped](#authorization-header-is-being-stripped). |
| ❌ **WordPress rejected the login** | A security or login plugin is interfering. | Temporarily disable security plugins and test again. |
| ❌ **Application passwords are disabled** | Turned off for the site or the user. | Check the Environment check section. |
| ❌ **REST API not found (404)** | Rewrite rules are stale. | **Settings → Permalinks → Save Changes**. |
| ❌ **Failed before reaching WordPress** | Staging password protection, firewall or CDN blocked it. | Turn off staging password protection or whitelist the request. |
| ❌ **Other HTTP error** | REST API blocked by a plugin. | Check Wordfence, Solid Security, AIOS or "Disable REST API" plugins. |

### Test it yourself with cURL

Every new password includes a ready-made command under **Test it yourself from a terminal (cURL)**:

```bash
curl --user "USERNAME:xxxx xxxx xxxx xxxx xxxx xxxx" https://example.com/wp-json/wp/v2/users/me
```

A working password returns your user details as JSON.

### Revoke passwords

- **Revoke** removes a single password. Any app using it stops working immediately.
- **Revoke all** removes every application password for the selected user.

Revoke keys you no longer use, and any key that was created but never shown.

---

## Recommended: use a dedicated user

An application password has **exactly the same permissions as the user it belongs to**. If you create it on an administrator account, anyone holding the key has full admin access through the API.

Safer setup:

1. **Users → Add New**. Create e.g. `api-bot` with the lowest role your app needs (often *Editor* or *Author*).
2. In **Tools → App Password Generator**, choose `api-bot` under **Managing passwords for**.
3. Generate the password there and use `api-bot` as the username in your app.

---

## Troubleshooting

### The password is never shown on the normal profile page

That's what this plugin is for. Use **Tools → App Password Generator** instead. On the profile page the key is shown by JavaScript after a REST API call, which security plugins, caching/minification, or JS errors often break.

### Authorization header is being stripped

Some Apache/LiteSpeed/CGI setups (including some shared hosts) drop the `Authorization` header before PHP sees it.

**Apache / LiteSpeed:** add this to the top of `.htaccess`, *above* `# BEGIN WordPress`:

```apache
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
</IfModule>
SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1
```

**Nginx + PHP-FPM:** add this inside your PHP `location` block, then reload Nginx:

```nginx
fastcgi_param HTTP_AUTHORIZATION $http_authorization;
```

Generate a new password with the test ticked to confirm the fix.

### "Application Passwords on this site: Disabled"

- **No HTTPS:** install an SSL certificate. On a *local or staging copy only*, you can add this to `wp-config.php` instead:
  ```php
  define( 'WP_ENVIRONMENT_TYPE', 'local' );
  ```
  Never use this on a live site.
- **A plugin disabled them:** many security plugins have an "Disable Application Passwords" option. Check Wordfence, Solid Security, AIOS, Hostinger Tools and similar.
- **Code snippet:** look for `add_filter( 'wp_is_application_passwords_available', '__return_false' )` in your theme's `functions.php` or a snippets plugin.

### Staging site with password protection

Hosting "staging password protection" (HTTP Basic Auth) uses the same `Authorization` header as application passwords, so the two conflict. Temporarily turn the protection off while using the API, or whitelist your app's IP.

### HTTPS shows ⓘ but my site uses HTTPS

If the site sits behind a proxy or CDN that doesn't pass the HTTPS flag to WordPress, this row can show a false warning. If **Application Passwords on this site** shows ✅, you can ignore it.

---

## Security

- Only users with `manage_options` (administrators) can open the tool.
- Creating passwords for another user also requires permission to edit that user.
- Every create/revoke action is protected by a WordPress nonce.
- The plain-text password is **never stored**. It exists only in the page response that shows it.
- Refreshing the page after creating a password will not create another one.
- The plugin only uses WordPress core's `WP_Application_Passwords` API. It has no custom authentication.

**Keep keys private.** Don't paste them into chats, screenshots, tickets or public repositories.

### Change who can use the tool

Add this to a small custom plugin or a snippets plugin:

```php
add_filter( 'apg_required_capability', function () {
	return 'edit_users'; // any capability you like
} );
```

---

## Uninstalling

Deactivate and delete the plugin from **Plugins**. It stores no settings, so nothing is left behind.

Passwords you created **keep working** after the plugin is deleted, because they belong to WordPress, not to this plugin. Revoke them under **Users → Profile → Application Passwords**, or reinstall this plugin to revoke them.

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## License

GPL-2.0-or-later. See [LICENSE](LICENSE).
