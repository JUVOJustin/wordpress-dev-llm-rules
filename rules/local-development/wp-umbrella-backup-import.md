# WP-Umbrella Backup Import in DDEV

Comprehensive guide for importing WP-Umbrella backups into DDEV local development environments.

## Overview

WP-Umbrella provides comprehensive backup solutions that can be imported into DDEV for local development. This process requires requesting a full backup download and properly importing both database and files while managing SMTP plugin conflicts.

## Prerequisites

- Active DDEV project with WordPress
- Access to WP-Umbrella dashboard
- WP-CLI installed in DDEV container

## Step 1: Request Full Backup

1. **Access WP-Umbrella Dashboard**: Navigate to your site's backup section
2. **Request Full Backup**: Click on "Download Full Backup" or equivalent option
3. **Wait for Notification**: You will receive both:
   - Email notification with download link
   - In-app notification in WP-Umbrella dashboard
   
⚠️ **Important**: The download link expires after approximately **15 minutes**. Download immediately upon receiving the notification.

## Step 2: Download and Extract Backup

1. **Download the backup archive** from the provided link
2. **Extract the archive** to access:
   - Database dump file (usually `.sql` or `.sql.gz`)
   - WordPress files directory

## Step 3: Disable SMTP Plugins

**Critical**: SMTP plugins must be disabled before import to prevent email sending attempts during development.

### Common SMTP Plugin Deactivation Commands

Execute these WP-CLI commands in your DDEV container:

```bash
# WP Mail SMTP
ddev wp plugin deactivate wp-mail-smtp

# Easy WP SMTP
ddev wp plugin deactivate easy-wp-smtp

# Post SMTP Mailer
ddev wp plugin deactivate post-smtp

# WP SMTP
ddev wp plugin deactivate wp-smtp

# SMTP Mailer
ddev wp plugin deactivate smtp-mailer

# Mailgun Official
ddev wp plugin deactivate mailgun

# SendGrid Email Delivery
ddev wp plugin deactivate sendgrid-email-delivery-simplified

# WP SES
ddev wp plugin deactivate wp-ses

# Check for any active SMTP-related plugins
ddev wp plugin list --status=active | grep -i smtp
```

## Step 4: Import Database

1. **Copy database file to DDEV project root**:
   ```bash
   cp /path/to/backup/database.sql .
   ```

2. **Import database using DDEV**:
   ```bash
   ddev import-db --src=database.sql
   ```

3. **Update site URLs**:
   ```bash
   ddev wp search-replace 'https://yoursite.com' 'https://yourproject.ddev.site'
   ddev wp search-replace 'http://yoursite.com' 'https://yourproject.ddev.site'
   ```

## Step 5: Import Files

1. **Backup existing wp-content** (optional):
   ```bash
   mv wp-content wp-content-backup
   ```

2. **Copy wp-content from backup**:
   ```bash
   cp -r /path/to/backup/wp-content .
   ```

3. **Set proper permissions**:
   ```bash
   ddev exec chown -R www-data:www-data wp-content
   ddev exec chmod -R 755 wp-content
   ```

## Step 6: Post-Import Configuration

### Update WordPress Configuration

1. **Flush rewrite rules**:
   ```bash
   ddev wp rewrite flush
   ```

2. **Update user passwords** for local development:
   ```bash
   ddev wp user update admin --user_pass=admin
   ```

3. **Verify site functionality**:
   ```bash
   ddev wp option get home
   ddev wp option get siteurl
   ```

### Development Environment Adjustments

1. **Enable debug mode** in `wp-config.php`:
   ```php
   define('WP_DEBUG', true);
   define('WP_DEBUG_LOG', true);
   define('WP_DEBUG_DISPLAY', false);
   ```

2. **Configure local mail handling**:
   ```bash
   # Install MailHog for email testing
   ddev get drud/ddev-mailhog
   ddev restart
   ```

## Troubleshooting

### Common Issues

**Database Import Errors**:
- Ensure database dump is not corrupted
- Check for character encoding issues
- Verify DDEV database service is running

**File Permission Issues**:
```bash
ddev exec find wp-content -type d -exec chmod 755 {} \;
ddev exec find wp-content -type f -exec chmod 644 {} \;
```

**Plugin Conflicts**:
```bash
# Deactivate all plugins temporarily
ddev wp plugin deactivate --all

# Reactivate plugins one by one for testing
ddev wp plugin activate plugin-name
```

**Memory or Timeout Issues**:
```bash
# Increase PHP limits in .ddev/config.yaml
php_version: "8.1"
webimage_extra_packages: [php8.1-gd]
```

### Verification Steps

1. **Check site accessibility**: Visit `https://yourproject.ddev.site`
2. **Verify admin access**: Login with updated credentials  
3. **Test core functionality**: Navigation, media, plugins
4. **Check email handling**: Confirm MailHog is capturing emails

## Security Considerations

- Never commit production database dumps to version control
- Remove or obfuscate sensitive data in development environment
- Use `.ddev/import-db/` directory for automatic cleanup of database files
- Regularly update local development dependencies

## Best Practices

- **Document the import process** specific to your project
- **Create import scripts** for recurring imports
- **Use staging environments** for testing before local import
- **Maintain separate development datasets** when possible
- **Regular backup cleanup** to prevent storage bloat