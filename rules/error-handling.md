# WordPress Error Handling Best Practices

Guidelines for implementing robust error handling in WordPress applications using WP_Exception, WP_Error, and wp_trigger_error.

## Error Handling Strategy Overview

Choose the appropriate method based on your application's architecture:

- **WP_Exception**: For strongly typed codebases where immediate error termination is required
- **WP_Error**: For REST API endpoints, AJAX handlers, and error accumulation scenarios  
- **wp_trigger_error**: For non-fatal error logging that respects environment-specific behavior

## WP_Exception for Strongly Typed Codebases

### When to Use WP_Exception

Use `WP_Exception` for:
- Immediate error termination with proper exception handling
- Modern PHP patterns (PHP 8.0+) in object-oriented code
- Complex business logic requiring precise error control

### Basic Implementation

```php
class User_Registration_Service {
    public function register_user(array $user_data): int {
        if (empty($user_data['username'])) {
            throw new WP_Exception(
                'missing_username',
                __('Username is required.', 'textdomain'),
                ['status' => 400]
            );
        }
        
        if (username_exists($user_data['username'])) {
            throw new WP_Exception(
                'username_exists',
                __('Username already taken.', 'textdomain'),
                ['status' => 409]
            );
        }
        
        $user_id = wp_create_user($user_data['username'], $user_data['password'], $user_data['email']);
        
        if (is_wp_error($user_id)) {
            throw new WP_Exception(
                'user_creation_failed',
                __('Failed to create user.', 'textdomain'),
                ['status' => 500]
            );
        }
        
        return $user_id;
    }
}
```

### Custom Exception Classes

```php
class Payment_Exception extends WP_Exception {
    private string $gateway_code;
    
    public function __construct(string $code, string $message, string $gateway_code, array $data = []) {
        $this->gateway_code = $gateway_code;
        parent::__construct($code, $message, $data);
    }
    
    public function get_gateway_code(): string {
        return $this->gateway_code;
    }
}

class Payment_Service {
    public function process_payment(array $payment_data): string {
        $response = $this->call_payment_gateway($payment_data);
        
        if (!$response['success']) {
            throw new Payment_Exception(
                'payment_declined',
                __('Payment declined.', 'textdomain'),
                $response['gateway_code'] ?? 'unknown'
            );
        }
        
        return $response['transaction_id'];
    }
}
```

## WP_Error for REST API and AJAX

### When to Use WP_Error

Use `WP_Error` for:
- REST API endpoint error responses
- AJAX request error handling  
- Form validation with multiple error accumulation
- WordPress hooks expecting WP_Error returns

### REST API Implementation

```php
class Posts_REST_Controller extends WP_REST_Controller {
    public function create_item($request) {
        $validation_errors = $this->validate_post_data($request);
        if (is_wp_error($validation_errors)) {
            return $validation_errors;
        }
        
        $post_id = wp_insert_post($this->prepare_item_for_database($request), true);
        
        if (is_wp_error($post_id)) {
            return new WP_Error(
                'post_creation_failed',
                __('Failed to create post.', 'textdomain'),
                ['status' => 500]
            );
        }
        
        return rest_ensure_response($this->prepare_item_for_response(get_post($post_id), $request));
    }
    
    private function validate_post_data(WP_REST_Request $request) {
        $errors = new WP_Error();
        
        if (empty($request->get_param('title'))) {
            $errors->add('missing_title', __('Title required.', 'textdomain'), ['field' => 'title']);
        }
        
        if (empty($request->get_param('content'))) {
            $errors->add('missing_content', __('Content required.', 'textdomain'), ['field' => 'content']);
        }
        
        return $errors->has_errors() ? $errors : true;
    }
}
```

### AJAX Implementation

WordPress `wp_send_json_error()` accepts WP_Error instances directly, automatically formatting error codes, messages, and data for JSON response.

```php
class Contact_Form_Handler {
    public function handle_contact_form(): void {
        $validation_result = $this->validate_contact_form($_POST);
        
        if (is_wp_error($validation_result)) {
            // wp_send_json_error() accepts WP_Error instances directly
            wp_send_json_error($validation_result);
        }
        
        $result = $this->process_contact_form($_POST);
        
        if (is_wp_error($result)) {
            // wp_send_json_error() accepts WP_Error instances directly
            wp_send_json_error($result);
        }
        
        wp_send_json_success(['message' => __('Message sent!', 'textdomain')]);
    }
    
    private function validate_contact_form(array $data) {
        $errors = new WP_Error();
        
        if (empty($data['name'])) {
            $errors->add('missing_name', __('Name required.', 'textdomain'), ['field' => 'name']);
        }
        
        if (empty($data['email']) || !is_email($data['email'])) {
            $errors->add('invalid_email', __('Valid email required.', 'textdomain'), ['field' => 'email']);
        }
        
        return $errors->has_errors() ? $errors : true;
    }
    
    // Optional: Custom error formatting for specific frontend requirements
    private function format_errors_for_frontend(WP_Error $error): array {
        $formatted = [];
        foreach ($error->get_error_codes() as $code) {
            $data = $error->get_error_data($code);
            $field = $data['field'] ?? 'general';
            $formatted[$field] = $error->get_error_messages($code);
        }
        return $formatted;
    }
}
```

To work with the wp error object you first need to check for the `success` property and then show the actual error message. Use this example method to get the error message.
```js
/**
 * Extract error message from WP_Error response
 * 
 * @param {Object} response - AJAX response object
 * @returns {string|null} Error message or null
 */
extractErrorMessage(response) {
    // Handle WP_Error format where data is an array of error objects
    if (!response.success && Array.isArray(response.data) && response.data.length > 0) {
        // Get the first error message
        return response.data[0].message || null;
    }
    
    return null;
}
```

## Error Logging with wp_trigger_error

### When to Use wp_trigger_error

Use `wp_trigger_error()` for:
- Non-fatal errors that shouldn't break user experience
- Development debugging respecting WP_DEBUG settings
- Deprecation notices and performance warnings

### Basic Implementation

```php
class Cache_Service {
    public function get_cached_data(string $cache_key) {
        if (empty($cache_key)) {
            wp_trigger_error(__FUNCTION__, __('Cache key empty.', 'textdomain'), E_USER_WARNING);
            return false;
        }
        
        $data = wp_cache_get($cache_key, 'custom_group');
        
        if (false === $data) {
            wp_trigger_error(
                __FUNCTION__,
                sprintf(__('Cache miss: %s', 'textdomain'), $cache_key),
                E_USER_NOTICE
            );
        }
        
        return $data;
    }
}
```

### Environment-Aware Error Handling

```php
class Error_Handler_Utility {
    public static function handle_error(string $context, string $message, $data = null, string $severity = 'warning'): void {
        $environment = wp_get_environment_type();
        
        if (defined('WP_DEBUG') && WP_DEBUG) {
            wp_trigger_error($context, $message, self::get_error_level($severity));
        }
        
        if (in_array($environment, ['development', 'staging'], true)) {
            error_log(sprintf('[%s] %s: %s', strtoupper($severity), $context, $message));
        }
    }
    
    private static function get_error_level(string $severity): int {
        return match ($severity) {
            'error' => E_USER_ERROR,
            'warning' => E_USER_WARNING,
            default => E_USER_NOTICE,
        };
    }
}
```

## Integration Patterns

### Combining Error Handling Methods

```php
class User_Profile_Service {
    public function update_profile(int $user_id, array $profile_data) {
        try {
            $this->validate_user_access($user_id); // Throws WP_Exception
            
            $validation_errors = $this->validate_profile_data($profile_data); // Returns WP_Error
            if (is_wp_error($validation_errors)) {
                return $validation_errors;
            }
            
            $this->process_profile_update($user_id, $profile_data); // Uses wp_trigger_error
            
            return ['success' => true, 'message' => __('Profile updated.', 'textdomain')];
            
        } catch (WP_Exception $e) {
            return new WP_Error($e->getErrorCode(), $e->getMessage(), ['status' => 403]);
        }
    }
    
    private function validate_user_access(int $user_id): void {
        if (!current_user_can('edit_user', $user_id)) {
            throw new WP_Exception('insufficient_permissions', __('Access denied.', 'textdomain'));
        }
    }
    
    private function validate_profile_data(array $data) {
        $errors = new WP_Error();
        
        if (empty($data['display_name'])) {
            $errors->add('missing_name', __('Display name required.', 'textdomain'), ['field' => 'display_name']);
        }
        
        return $errors->has_errors() ? $errors : true;
    }
    
    private function process_profile_update(int $user_id, array $data): void {
        foreach ($data as $key => $value) {
            if (!update_user_meta($user_id, $key, $value)) {
                wp_trigger_error(__FUNCTION__, sprintf('Meta update failed: %s', $key), E_USER_WARNING);
            }
        }
    }
}
```

## Decision Matrix

| Scenario | WP_Exception | WP_Error | wp_trigger_error |
|----------|:------------:|:--------:|:----------------:|
| REST API validation | ❌ | ✅ | ❌ |
| AJAX form processing | ❌ | ✅ | ❌ |
| Service layer logic | ✅ | ❌ | ❌ |
| Configuration issues | ❌ | ❌ | ✅ |
| Development debugging | ❌ | ❌ | ✅ |
| Critical failures | ✅ | ❌ | ❌ |
| Multiple field errors | ❌ | ✅ | ❌ |
| Performance warnings | ❌ | ❌ | ✅ |

## Key Guidelines

1. **WP_Exception**: Immediate termination in strongly typed code
2. **WP_Error**: Error accumulation for APIs and forms
3. **wp_trigger_error**: Non-fatal logging respecting environment
4. Always provide context and meaningful error codes
5. Use internationalization for user-facing messages
6. Include relevant debug data in error objects
