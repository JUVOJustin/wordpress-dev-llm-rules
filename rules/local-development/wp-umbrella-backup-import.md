# WP-Umbrella Backup Import in DDEV

Process for importing WP-Umbrella backups into DDEV local development environments.

## Prerequisites

- Active DDEV project with WordPress
- Access to WP-Umbrella dashboard
- WP-CLI installed in DDEV container

## Import Process

### 1. Request and Download Backup

1. Navigate to WP-Umbrella dashboard → Backups → "Download Full Backup"
2. Download from email/dashboard notification immediately (⚠️ **15-minute expiration**)
3. Extract archive to access database dump and WordPress files

### 2. Disable SMTP Plugins (Critical)

Deactivate SMTP plugins before import to prevent email sending:

```bash
# Common SMTP plugins
ddev wp plugin deactivate wp-mail-smtp easy-wp-smtp post-smtp wp-smtp smtp-mailer mailgun sendgrid-email-delivery-simplified wp-ses

# Verify no SMTP plugins active
ddev wp plugin list --status=active | grep -i smtp
```

### 3. Import Database

```bash
# Import database
ddev import-db --src=/path/to/backup/database.sql

# Update URLs
ddev wp search-replace 'https://production-site.com' 'https://yourproject.ddev.site'
ddev wp search-replace 'http://production-site.com' 'https://yourproject.ddev.site'
```

### 4. Import Files

```bash
# Backup existing wp-content (optional)
mv wp-content wp-content-backup

# Copy wp-content from backup
cp -r /path/to/backup/wp-content .

# Set permissions
ddev exec chown -R www-data:www-data wp-content
ddev exec chmod -R 755 wp-content
```

### 5. Post-Import Configuration

```bash
# Flush rewrite rules
ddev wp rewrite flush

# Update admin password
ddev wp user update admin --user_pass=admin

# Verify URLs
ddev wp option get home
ddev wp option get siteurl
```

### 6. Development Setup

```bash
# Enable debug mode in wp-config.php
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', false);

# Install MailHog for email testing
ddev get drud/ddev-mailhog
ddev restart
```

## Troubleshooting

**Database Issues**: Check dump corruption, encoding, DDEV database service status

**Permission Issues**:
```bash
ddev exec find wp-content -type d -exec chmod 755 {} \;
ddev exec find wp-content -type f -exec chmod 644 {} \;
```

**Plugin Conflicts**:
```bash
ddev wp plugin deactivate --all  # Temporarily disable all plugins
```

**Memory/Timeout**: Increase PHP limits in `.ddev/config.yaml`

## Security Notes

- Never commit production database dumps to version control
- Use `.ddev/import-db/` directory for automatic cleanup
- Remove sensitive data from development environment