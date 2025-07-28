# WP-Umbrella Backup Import in DDEV

Process for importing WP-Umbrella backups into DDEV local development environments.

## Download and Extract Backup

Move the downloaded `.tar.gz` backup file to your DDEV project folder and extract:

```bash
tar -xvzf filename.tar.gz
```

## Disable SMTP Plugins

Disable SMTP plugins before import to prevent email sending attempts during development:

```bash
# Common SMTP plugins
ddev wp plugin deactivate wp-mail-smtp
ddev wp plugin deactivate easy-wp-smtp
ddev wp plugin deactivate post-smtp
ddev wp plugin deactivate wp-smtp
ddev wp plugin deactivate smtp-mailer
ddev wp plugin deactivate mailgun
ddev wp plugin deactivate sendgrid-email-delivery-simplified
ddev wp plugin deactivate wp-ses
```

## Import Database

Ensure your project is configured (`ddev config`) and running (`ddev start`). Enter the DDEV container and import all SQL files:

```bash
ddev ssh
cd umb_database
for file in *.sql; do mysql -u db -p'db' db < "$file"; done
exit
```

## WordPress Configuration

Edit `wp-config.php` and remove old database credentials (`DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`) and debug settings (`WP_DEBUG`, `WP_ENVIRONMENT_TYPE`) as DDEV manages these.

Add DDEV configuration if not present:

```php
// Include for ddev-managed settings in wp-config-ddev.php.
$ddev_settings = dirname(__FILE__) . '/wp-config-ddev.php';
if (is_readable($ddev_settings) && !defined('DB_USER')) {
    require_once($ddev_settings);
}
```

Add environment type for local development:

```php
define('WP_ENVIRONMENT_TYPE', 'local');
```

If you changed the DB prefix, move it from `wp-config.php` to `wp-config-ddev.php` and remove the header comment in `wp-config-ddev.php` to prevent DDEV from resetting changes.

## Update URLs

```bash
ddev wp search-replace "my-website.com" "${DDEV_HOSTNAME}"
```

## Verification

Test the installation:

```bash
ddev wp
```

If no errors are returned, the import was successful.