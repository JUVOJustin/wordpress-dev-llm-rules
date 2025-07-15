# WordPress Error Handling Best Practices

Comprehensive guidelines for implementing robust error handling in WordPress applications using WP_Exception, WP_Error, and wp_trigger_error.

## Error Handling Strategy Overview

WordPress provides multiple error handling mechanisms designed for different contexts and use cases. Choose the appropriate method based on your application's architecture and the specific scenario:

- **WP_Exception**: For strongly typed, object-oriented codebases where immediate error termination is required
- **WP_Error**: For REST API endpoints, AJAX handlers, and scenarios requiring error accumulation
- **wp_trigger_error**: For non-fatal error logging that respects environment-specific behavior

## WP_Exception for Strongly Typed Codebases

### When to Use WP_Exception

Use `WP_Exception` in strongly typed, object-oriented WordPress applications where:
- You need immediate error termination with proper exception handling
- Working with modern PHP patterns (PHP 8.0+)
- Building complex business logic that requires precise error control
- Implementing services, repositories, or domain-specific classes

### Basic WP_Exception Implementation

```php
<?php
/**
 * Service class demonstrating WP_Exception usage.
 */
class User_Registration_Service {
    
    /**
     * Register a new user with validation.
     *
     * @param array $user_data User registration data.
     * @return int User ID on success.
     * @throws WP_Exception When validation fails or user creation fails.
     */
    public function register_user(array $user_data): int {
        // Validate required fields
        if (empty($user_data['username'])) {
            throw new WP_Exception(
                'missing_username',
                __('Username is required for registration.', 'textdomain'),
                ['status' => 400]
            );
        }
        
        if (empty($user_data['email']) || !is_email($user_data['email'])) {
            throw new WP_Exception(
                'invalid_email',
                __('A valid email address is required.', 'textdomain'),
                ['status' => 400]
            );
        }
        
        // Check for existing username
        if (username_exists($user_data['username'])) {
            throw new WP_Exception(
                'username_exists',
                __('This username is already taken.', 'textdomain'),
                ['status' => 409]
            );
        }
        
        // Attempt user creation
        $user_id = wp_create_user(
            $user_data['username'],
            $user_data['password'],
            $user_data['email']
        );
        
        if (is_wp_error($user_id)) {
            throw new WP_Exception(
                'user_creation_failed',
                sprintf(
                    /* translators: %s: WordPress error message */
                    __('Failed to create user: %s', 'textdomain'),
                    $user_id->get_error_message()
                ),
                ['status' => 500, 'wp_error' => $user_id]
            );
        }
        
        return $user_id;
    }
}
```

### Custom WP_Exception Extensions

Create domain-specific exception classes for better error handling:

```php
<?php
/**
 * Custom exception for payment processing errors.
 */
class Payment_Exception extends WP_Exception {
    
    /**
     * Payment gateway error code.
     *
     * @var string
     */
    private string $gateway_code;
    
    /**
     * Constructor.
     *
     * @param string $code Error code.
     * @param string $message Error message.
     * @param string $gateway_code Payment gateway specific error code.
     * @param array  $data Additional error data.
     */
    public function __construct(string $code, string $message, string $gateway_code, array $data = []) {
        $this->gateway_code = $gateway_code;
        $data['gateway_code'] = $gateway_code;
        
        parent::__construct($code, $message, $data);
    }
    
    /**
     * Get the payment gateway error code.
     *
     * @return string
     */
    public function get_gateway_code(): string {
        return $this->gateway_code;
    }
}

/**
 * Payment service using custom exception.
 */
class Payment_Service {
    
    /**
     * Process payment with comprehensive error handling.
     *
     * @param array $payment_data Payment information.
     * @return string Transaction ID.
     * @throws Payment_Exception When payment processing fails.
     */
    public function process_payment(array $payment_data): string {
        try {
            // Simulate payment gateway API call
            $response = $this->call_payment_gateway($payment_data);
            
            if (!$response['success']) {
                throw new Payment_Exception(
                    'payment_declined',
                    __('Payment was declined by the gateway.', 'textdomain'),
                    $response['gateway_code'] ?? 'unknown',
                    ['gateway_response' => $response]
                );
            }
            
            return $response['transaction_id'];
            
        } catch (Exception $e) {
            // Convert any unexpected exceptions to Payment_Exception
            throw new Payment_Exception(
                'payment_gateway_error',
                sprintf(
                    /* translators: %s: Original error message */
                    __('Payment gateway error: %s', 'textdomain'),
                    $e->getMessage()
                ),
                'gateway_exception',
                ['original_exception' => $e]
            );
        }
    }
}
```

## WP_Error for REST API and AJAX

### When to Use WP_Error

Use `WP_Error` for:
- REST API endpoint error responses
- AJAX request error handling
- Form validation with multiple error accumulation
- Any scenario where you need to collect and return multiple errors
- WordPress hooks and filters that expect WP_Error returns

### REST API Error Handling

```php
<?php
/**
 * REST API controller with comprehensive WP_Error usage.
 */
class Posts_REST_Controller extends WP_REST_Controller {
    
    /**
     * Create a new post with validation.
     *
     * @param WP_REST_Request $request Full details about the request.
     * @return WP_REST_Response|WP_Error Response object on success, error on failure.
     */
    public function create_item($request) {
        // Validate the request and accumulate errors
        $validation_errors = $this->validate_post_data($request);
        
        if (is_wp_error($validation_errors)) {
            return $validation_errors;
        }
        
        // Additional business logic validation
        $business_errors = $this->validate_business_rules($request);
        if (is_wp_error($business_errors)) {
            return $business_errors;
        }
        
        // Create the post
        $post_data = $this->prepare_item_for_database($request);
        $post_id = wp_insert_post($post_data, true);
        
        if (is_wp_error($post_id)) {
            return new WP_Error(
                'post_creation_failed',
                __('Failed to create the post.', 'textdomain'),
                ['status' => 500, 'details' => $post_id->get_error_messages()]
            );
        }
        
        $post = get_post($post_id);
        $response = $this->prepare_item_for_response($post, $request);
        
        return rest_ensure_response($response);
    }
    
    /**
     * Validate post data and accumulate multiple errors.
     *
     * @param WP_REST_Request $request Request object.
     * @return WP_Error|true WP_Error if validation fails, true on success.
     */
    private function validate_post_data(WP_REST_Request $request) {
        $errors = new WP_Error();
        
        // Validate title
        $title = $request->get_param('title');
        if (empty($title)) {
            $errors->add(
                'missing_title',
                __('Post title is required.', 'textdomain'),
                ['field' => 'title']
            );
        } elseif (strlen($title) > 200) {
            $errors->add(
                'title_too_long',
                __('Post title must be 200 characters or less.', 'textdomain'),
                ['field' => 'title', 'max_length' => 200]
            );
        }
        
        // Validate content
        $content = $request->get_param('content');
        if (empty($content)) {
            $errors->add(
                'missing_content',
                __('Post content is required.', 'textdomain'),
                ['field' => 'content']
            );
        }
        
        // Validate status
        $status = $request->get_param('status');
        $allowed_statuses = ['draft', 'publish', 'private'];
        if (!empty($status) && !in_array($status, $allowed_statuses, true)) {
            $errors->add(
                'invalid_status',
                sprintf(
                    /* translators: %s: Comma-separated list of allowed statuses */
                    __('Invalid post status. Allowed values: %s', 'textdomain'),
                    implode(', ', $allowed_statuses)
                ),
                ['field' => 'status', 'allowed_values' => $allowed_statuses]
            );
        }
        
        // Validate categories
        $categories = $request->get_param('categories');
        if (!empty($categories) && is_array($categories)) {
            foreach ($categories as $category_id) {
                if (!term_exists($category_id, 'category')) {
                    $errors->add(
                        'invalid_category',
                        sprintf(
                            /* translators: %d: Category ID */
                            __('Category with ID %d does not exist.', 'textdomain'),
                            $category_id
                        ),
                        ['field' => 'categories', 'invalid_id' => $category_id]
                    );
                }
            }
        }
        
        // Return errors if any exist
        if ($errors->has_errors()) {
            return $errors;
        }
        
        return true;
    }
    
    /**
     * Validate business rules.
     *
     * @param WP_REST_Request $request Request object.
     * @return WP_Error|true WP_Error if validation fails, true on success.
     */
    private function validate_business_rules(WP_REST_Request $request) {
        $errors = new WP_Error();
        
        // Check user permissions
        if (!current_user_can('publish_posts') && 'publish' === $request->get_param('status')) {
            $errors->add(
                'insufficient_permissions',
                __('You do not have permission to publish posts.', 'textdomain'),
                ['status' => 403]
            );
        }
        
        // Check for duplicate content
        $title = $request->get_param('title');
        if (!empty($title)) {
            $existing_post = get_page_by_title($title, OBJECT, 'post');
            if ($existing_post) {
                $errors->add(
                    'duplicate_title',
                    __('A post with this title already exists.', 'textdomain'),
                    ['field' => 'title', 'existing_post_id' => $existing_post->ID]
                );
            }
        }
        
        return $errors->has_errors() ? $errors : true;
    }
}
```

### AJAX Error Handling

```php
<?php
/**
 * AJAX handler with WP_Error for form processing.
 */
class Contact_Form_Handler {
    
    /**
     * Initialize AJAX hooks.
     */
    public function init(): void {
        add_action('wp_ajax_submit_contact_form', [$this, 'handle_contact_form']);
        add_action('wp_ajax_nopriv_submit_contact_form', [$this, 'handle_contact_form']);
    }
    
    /**
     * Handle contact form submission.
     */
    public function handle_contact_form(): void {
        // Verify nonce
        if (!wp_verify_nonce($_POST['nonce'] ?? '', 'contact_form_nonce')) {
            wp_send_json_error([
                'message' => __('Security verification failed.', 'textdomain'),
                'code' => 'invalid_nonce'
            ]);
        }
        
        // Validate form data
        $validation_result = $this->validate_contact_form($_POST);
        
        if (is_wp_error($validation_result)) {
            // Send structured error response with all validation errors
            wp_send_json_error([
                'message' => __('Please correct the errors below.', 'textdomain'),
                'code' => 'validation_failed',
                'errors' => $this->format_errors_for_frontend($validation_result)
            ]);
        }
        
        // Process the form
        $result = $this->process_contact_form($_POST);
        
        if (is_wp_error($result)) {
            wp_send_json_error([
                'message' => $result->get_error_message(),
                'code' => $result->get_error_code()
            ]);
        }
        
        wp_send_json_success([
            'message' => __('Your message has been sent successfully!', 'textdomain')
        ]);
    }
    
    /**
     * Validate contact form data.
     *
     * @param array $data Form data.
     * @return WP_Error|true WP_Error if validation fails, true on success.
     */
    private function validate_contact_form(array $data) {
        $errors = new WP_Error();
        
        // Validate name
        $name = sanitize_text_field($data['name'] ?? '');
        if (empty($name)) {
            $errors->add(
                'missing_name',
                __('Name is required.', 'textdomain'),
                ['field' => 'name']
            );
        } elseif (strlen($name) < 2) {
            $errors->add(
                'name_too_short',
                __('Name must be at least 2 characters long.', 'textdomain'),
                ['field' => 'name']
            );
        }
        
        // Validate email
        $email = sanitize_email($data['email'] ?? '');
        if (empty($email)) {
            $errors->add(
                'missing_email',
                __('Email address is required.', 'textdomain'),
                ['field' => 'email']
            );
        } elseif (!is_email($email)) {
            $errors->add(
                'invalid_email',
                __('Please enter a valid email address.', 'textdomain'),
                ['field' => 'email']
            );
        }
        
        // Validate subject
        $subject = sanitize_text_field($data['subject'] ?? '');
        if (empty($subject)) {
            $errors->add(
                'missing_subject',
                __('Subject is required.', 'textdomain'),
                ['field' => 'subject']
            );
        }
        
        // Validate message
        $message = sanitize_textarea_field($data['message'] ?? '');
        if (empty($message)) {
            $errors->add(
                'missing_message',
                __('Message is required.', 'textdomain'),
                ['field' => 'message']
            );
        } elseif (strlen($message) < 10) {
            $errors->add(
                'message_too_short',
                __('Message must be at least 10 characters long.', 'textdomain'),
                ['field' => 'message']
            );
        }
        
        return $errors->has_errors() ? $errors : true;
    }
    
    /**
     * Format WP_Error for frontend consumption.
     *
     * @param WP_Error $error WP_Error object.
     * @return array Formatted errors by field.
     */
    private function format_errors_for_frontend(WP_Error $error): array {
        $formatted_errors = [];
        
        foreach ($error->get_error_codes() as $code) {
            $messages = $error->get_error_messages($code);
            $data = $error->get_error_data($code);
            $field = $data['field'] ?? 'general';
            
            if (!isset($formatted_errors[$field])) {
                $formatted_errors[$field] = [];
            }
            
            $formatted_errors[$field] = array_merge($formatted_errors[$field], $messages);
        }
        
        return $formatted_errors;
    }
}
```

## Error Logging with wp_trigger_error

### When to Use wp_trigger_error

Use `wp_trigger_error()` for:
- Non-fatal errors that shouldn't break user experience in production
- Development debugging that respects WP_DEBUG settings
- Logging errors that need developer attention without affecting end users
- Deprecation notices and warnings
- Performance or configuration issues

### Basic wp_trigger_error Usage

```php
<?php
/**
 * Service class demonstrating wp_trigger_error usage.
 */
class Cache_Service {
    
    /**
     * Get cached data with error logging.
     *
     * @param string $cache_key Cache key.
     * @return mixed|false Cached data or false on failure.
     */
    public function get_cached_data(string $cache_key) {
        if (empty($cache_key)) {
            wp_trigger_error(
                __FUNCTION__,
                __('Cache key cannot be empty.', 'textdomain'),
                E_USER_WARNING
            );
            return false;
        }
        
        $data = wp_cache_get($cache_key, 'custom_group');
        
        if (false === $data) {
            // Log cache miss for debugging (non-fatal)
            wp_trigger_error(
                __FUNCTION__,
                sprintf(
                    /* translators: %s: Cache key */
                    __('Cache miss for key: %s', 'textdomain'),
                    $cache_key
                ),
                E_USER_NOTICE
            );
        }
        
        return $data;
    }
    
    /**
     * Set cached data with validation.
     *
     * @param string $cache_key Cache key.
     * @param mixed  $data Data to cache.
     * @param int    $expiration Cache expiration time.
     * @return bool True on success, false on failure.
     */
    public function set_cached_data(string $cache_key, $data, int $expiration = 3600): bool {
        if (empty($cache_key)) {
            wp_trigger_error(
                __FUNCTION__,
                __('Cache key cannot be empty.', 'textdomain'),
                E_USER_WARNING
            );
            return false;
        }
        
        if ($expiration < 0) {
            wp_trigger_error(
                __FUNCTION__,
                sprintf(
                    /* translators: %d: Provided expiration time */
                    __('Invalid cache expiration time: %d. Using default.', 'textdomain'),
                    $expiration
                ),
                E_USER_WARNING
            );
            $expiration = 3600;
        }
        
        $result = wp_cache_set($cache_key, $data, 'custom_group', $expiration);
        
        if (!$result) {
            wp_trigger_error(
                __FUNCTION__,
                sprintf(
                    /* translators: %s: Cache key */
                    __('Failed to set cache for key: %s', 'textdomain'),
                    $cache_key
                ),
                E_USER_WARNING
            );
        }
        
        return $result;
    }
}

/**
 * Configuration service with environment-aware error handling.
 */
class Configuration_Service {
    
    /**
     * Get configuration value with fallback and logging.
     *
     * @param string $option_name Option name.
     * @param mixed  $default Default value.
     * @return mixed Configuration value.
     */
    public function get_config(string $option_name, $default = null) {
        $value = get_option($option_name, $default);
        
        // Log when falling back to default values
        if ($value === $default && null !== $default) {
            wp_trigger_error(
                __FUNCTION__,
                sprintf(
                    /* translators: 1: Option name, 2: Default value */
                    __('Option "%1$s" not found, using default: %2$s', 'textdomain'),
                    $option_name,
                    is_scalar($default) ? $default : gettype($default)
                ),
                E_USER_NOTICE
            );
        }
        
        return $value;
    }
    
    /**
     * Validate plugin requirements.
     */
    public function validate_requirements(): void {
        // Check PHP version
        if (version_compare(PHP_VERSION, '8.0', '<')) {
            wp_trigger_error(
                __FUNCTION__,
                sprintf(
                    /* translators: 1: Current PHP version, 2: Required PHP version */
                    __('PHP version %1$s detected. PHP %2$s or higher is recommended.', 'textdomain'),
                    PHP_VERSION,
                    '8.0'
                ),
                E_USER_WARNING
            );
        }
        
        // Check required extensions
        $required_extensions = ['mbstring', 'curl', 'json'];
        foreach ($required_extensions as $extension) {
            if (!extension_loaded($extension)) {
                wp_trigger_error(
                    __FUNCTION__,
                    sprintf(
                        /* translators: %s: Extension name */
                        __('Required PHP extension "%s" is not loaded.', 'textdomain'),
                        $extension
                    ),
                    E_USER_ERROR
                );
            }
        }
        
        // Check memory limit
        $memory_limit = ini_get('memory_limit');
        $memory_bytes = $this->convert_to_bytes($memory_limit);
        $recommended_bytes = 256 * 1024 * 1024; // 256MB
        
        if ($memory_bytes < $recommended_bytes) {
            wp_trigger_error(
                __FUNCTION__,
                sprintf(
                    /* translators: 1: Current memory limit, 2: Recommended memory limit */
                    __('Memory limit is %1$s. %2$s or higher is recommended for optimal performance.', 'textdomain'),
                    $memory_limit,
                    '256M'
                ),
                E_USER_WARNING
            );
        }
    }
    
    /**
     * Convert memory limit string to bytes.
     *
     * @param string $memory_limit Memory limit string.
     * @return int Memory limit in bytes.
     */
    private function convert_to_bytes(string $memory_limit): int {
        $memory_limit = trim($memory_limit);
        $last_char = strtolower(substr($memory_limit, -1));
        $number = (int) substr($memory_limit, 0, -1);
        
        switch ($last_char) {
            case 'g':
                return $number * 1024 * 1024 * 1024;
            case 'm':
                return $number * 1024 * 1024;
            case 'k':
                return $number * 1024;
            default:
                return (int) $memory_limit;
        }
    }
}
```

## Error Handling Integration Patterns

### Combining Error Handling Methods

```php
<?php
/**
 * Service demonstrating integrated error handling approach.
 */
class User_Profile_Service {
    
    /**
     * Update user profile with comprehensive error handling.
     *
     * @param int   $user_id User ID.
     * @param array $profile_data Profile data.
     * @return WP_Error|array Success response or error.
     */
    public function update_profile(int $user_id, array $profile_data) {
        try {
            // Use WP_Exception for critical validations
            $this->validate_user_access($user_id);
            
            // Use WP_Error for field validation (allows accumulation)
            $validation_errors = $this->validate_profile_data($profile_data);
            if (is_wp_error($validation_errors)) {
                return $validation_errors;
            }
            
            // Process the update
            $result = $this->process_profile_update($user_id, $profile_data);
            
            return [
                'success' => true,
                'message' => __('Profile updated successfully.', 'textdomain'),
                'user_id' => $user_id
            ];
            
        } catch (WP_Exception $e) {
            // Convert WP_Exception to WP_Error for consistent API response
            return new WP_Error(
                $e->getErrorCode(),
                $e->getMessage(),
                array_merge(
                    ['status' => 403],
                    $e->getErrorData()
                )
            );
        }
    }
    
    /**
     * Validate user access using WP_Exception for immediate termination.
     *
     * @param int $user_id User ID.
     * @throws WP_Exception When access is denied.
     */
    private function validate_user_access(int $user_id): void {
        if (!get_userdata($user_id)) {
            throw new WP_Exception(
                'user_not_found',
                __('User not found.', 'textdomain'),
                ['user_id' => $user_id]
            );
        }
        
        $current_user_id = get_current_user_id();
        if (0 === $current_user_id) {
            throw new WP_Exception(
                'user_not_logged_in',
                __('You must be logged in to update profiles.', 'textdomain')
            );
        }
        
        if ($current_user_id !== $user_id && !current_user_can('edit_users')) {
            throw new WP_Exception(
                'insufficient_permissions',
                __('You do not have permission to edit this profile.', 'textdomain'),
                ['required_capability' => 'edit_users']
            );
        }
    }
    
    /**
     * Validate profile data using WP_Error for field accumulation.
     *
     * @param array $profile_data Profile data.
     * @return WP_Error|true WP_Error if validation fails, true on success.
     */
    private function validate_profile_data(array $profile_data) {
        $errors = new WP_Error();
        
        // Validate display name
        $display_name = sanitize_text_field($profile_data['display_name'] ?? '');
        if (empty($display_name)) {
            $errors->add(
                'missing_display_name',
                __('Display name is required.', 'textdomain'),
                ['field' => 'display_name']
            );
        }
        
        // Validate bio length
        $bio = sanitize_textarea_field($profile_data['bio'] ?? '');
        if (strlen($bio) > 500) {
            $errors->add(
                'bio_too_long',
                __('Bio must be 500 characters or less.', 'textdomain'),
                ['field' => 'bio', 'max_length' => 500]
            );
        }
        
        return $errors->has_errors() ? $errors : true;
    }
    
    /**
     * Process profile update with error logging.
     *
     * @param int   $user_id User ID.
     * @param array $profile_data Profile data.
     * @return bool Success status.
     */
    private function process_profile_update(int $user_id, array $profile_data): bool {
        // Update user meta with error logging
        foreach ($profile_data as $meta_key => $meta_value) {
            $result = update_user_meta($user_id, $meta_key, $meta_value);
            
            if (false === $result) {
                // Use wp_trigger_error for non-critical meta update failures
                wp_trigger_error(
                    __FUNCTION__,
                    sprintf(
                        /* translators: 1: Meta key, 2: User ID */
                        __('Failed to update user meta "%1$s" for user %2$d', 'textdomain'),
                        $meta_key,
                        $user_id
                    ),
                    E_USER_WARNING
                );
            }
        }
        
        return true;
    }
}
```

## Environment-Specific Error Handling

### Development vs Production Behavior

```php
<?php
/**
 * Environment-aware error handling utility.
 */
class Error_Handler_Utility {
    
    /**
     * Handle errors based on environment type.
     *
     * @param string $context Function or context name.
     * @param string $message Error message.
     * @param mixed  $data Additional error data.
     * @param string $severity Error severity level.
     */
    public static function handle_error(
        string $context,
        string $message,
        $data = null,
        string $severity = 'warning'
    ): void {
        $environment = wp_get_environment_type();
        
        // Always log errors when WP_DEBUG is enabled
        if (defined('WP_DEBUG') && WP_DEBUG) {
            $error_level = self::get_error_level($severity);
            wp_trigger_error($context, $message, $error_level);
        }
        
        // Enhanced logging for development environments
        if (in_array($environment, ['development', 'staging'], true)) {
            error_log(sprintf(
                '[%s] %s: %s %s',
                strtoupper($severity),
                $context,
                $message,
                $data ? '| Data: ' . wp_json_encode($data) : ''
            ));
        }
        
        // Send to external logging service in production
        if ('production' === $environment && 'error' === $severity) {
            self::send_to_external_logger($context, $message, $data);
        }
    }
    
    /**
     * Get PHP error level from severity string.
     *
     * @param string $severity Severity level.
     * @return int PHP error level constant.
     */
    private static function get_error_level(string $severity): int {
        switch ($severity) {
            case 'error':
                return E_USER_ERROR;
            case 'warning':
                return E_USER_WARNING;
            case 'notice':
            default:
                return E_USER_NOTICE;
        }
    }
    
    /**
     * Send error to external logging service.
     *
     * @param string $context Function or context name.
     * @param string $message Error message.
     * @param mixed  $data Additional error data.
     */
    private static function send_to_external_logger(string $context, string $message, $data): void {
        // Implementation would depend on your logging service
        // Example: Sentry, LogRocket, or custom API
        
        // Log locally as fallback
        error_log(sprintf(
            '[PRODUCTION ERROR] %s: %s',
            $context,
            $message
        ));
    }
}

/**
 * Usage example with environment-aware error handling.
 */
class API_Service {
    
    /**
     * Make external API call with comprehensive error handling.
     *
     * @param string $endpoint API endpoint.
     * @param array  $data Request data.
     * @return array|WP_Error API response or error.
     */
    public function make_api_call(string $endpoint, array $data = []) {
        $response = wp_remote_post($endpoint, [
            'body' => wp_json_encode($data),
            'headers' => ['Content-Type' => 'application/json'],
            'timeout' => 30,
        ]);
        
        if (is_wp_error($response)) {
            Error_Handler_Utility::handle_error(
                __FUNCTION__,
                sprintf(
                    /* translators: 1: Endpoint URL, 2: Error message */
                    __('API request failed for %1$s: %2$s', 'textdomain'),
                    $endpoint,
                    $response->get_error_message()
                ),
                ['endpoint' => $endpoint, 'wp_error' => $response],
                'error'
            );
            
            return $response;
        }
        
        $response_code = wp_remote_retrieve_response_code($response);
        if ($response_code >= 400) {
            $error_message = sprintf(
                /* translators: 1: HTTP status code, 2: Endpoint URL */
                __('API returned error status %1$d for %2$s', 'textdomain'),
                $response_code,
                $endpoint
            );
            
            Error_Handler_Utility::handle_error(
                __FUNCTION__,
                $error_message,
                [
                    'endpoint' => $endpoint,
                    'status_code' => $response_code,
                    'response_body' => wp_remote_retrieve_body($response)
                ],
                'warning'
            );
            
            return new WP_Error(
                'api_error',
                $error_message,
                ['status' => $response_code]
            );
        }
        
        $body = wp_remote_retrieve_body($response);
        $decoded = json_decode($body, true);
        
        if (null === $decoded && JSON_ERROR_NONE !== json_last_error()) {
            Error_Handler_Utility::handle_error(
                __FUNCTION__,
                sprintf(
                    /* translators: %s: JSON error message */
                    __('Failed to decode API response: %s', 'textdomain'),
                    json_last_error_msg()
                ),
                ['endpoint' => $endpoint, 'raw_body' => $body],
                'warning'
            );
            
            return new WP_Error(
                'json_decode_error',
                __('Invalid JSON response from API', 'textdomain')
            );
        }
        
        return $decoded;
    }
}
```

## Best Practices Summary

### Error Handling Decision Matrix

| Scenario | Use WP_Exception | Use WP_Error | Use wp_trigger_error |
|----------|------------------|--------------|---------------------|
| REST API endpoint validation | ❌ | ✅ | ❌ |
| AJAX form processing | ❌ | ✅ | ❌ |
| Service layer business logic | ✅ | ❌ | ❌ |
| Non-fatal configuration issues | ❌ | ❌ | ✅ |
| Development debugging | ❌ | ❌ | ✅ |
| Critical system failures | ✅ | ❌ | ❌ |
| Multiple validation errors | ❌ | ✅ | ❌ |
| Performance warnings | ❌ | ❌ | ✅ |

### Key Guidelines

1. **Use WP_Exception** for immediate error termination in strongly typed code
2. **Use WP_Error** when you need to accumulate multiple errors or return errors to REST/AJAX
3. **Use wp_trigger_error** for non-fatal issues that should be logged but not break user experience
4. **Always provide context** in error messages with specific details about what failed
5. **Include error codes** that are consistent and meaningful for programmatic handling
6. **Respect environment types** - be more verbose in development, quieter in production
7. **Provide internationalized error messages** using WordPress translation functions
8. **Include relevant data** in error objects to help with debugging and user feedback