# WordPress REST API Schema Validation: Strict Guidelines

## Mandatory Implementation Rules

**ALWAYS use WP_REST_Controller** - Never implement REST endpoints without extending this class.

**ALWAYS use both validation functions** in this exact order:
1. `rest_validate_value_from_schema($value, $schema, $param)`
2. `rest_sanitize_value_from_schema($value, $schema, $param)` (only if validation returns `true`)

**SECURITY WARNING**: Skipping validation caused CVE-2017-1001000. Both functions are required to prevent injection attacks.

## Required Controller Pattern

```php
class My_Controller extends WP_REST_Controller {
    
    protected $namespace = 'plugin-name/v1';
    protected $rest_base = 'resource';
    
    public function register_routes() {
        register_rest_route($this->namespace, '/' . $this->rest_base, array(
            array(
                'methods'             => WP_REST_Server::CREATABLE,
                'callback'            => array($this, 'create_item'),
                'permission_callback' => array($this, 'create_item_permissions_check'),
                // get_endpoint_args_for_item_schema() automatically converts your schema
                // into argument definitions with built-in validate_callback and sanitize_callback
                // This eliminates the need to manually specify validation/sanitization
                'args'                => $this->get_endpoint_args_for_item_schema(WP_REST_Server::CREATABLE),
            ),
            // 'schema' callback allows WordPress to auto-generate API documentation
            // and provides schema information to clients via OPTIONS requests
            'schema' => array($this, 'get_public_item_schema'),
        ));
    }
    
    public function get_item_schema() {
        // Cache schema to avoid regenerating it on multiple calls
        if ($this->schema) {
            // add_additional_fields_schema() merges custom fields added via register_rest_field()
            // This allows plugins to extend your schema without modifying core code
            return $this->add_additional_fields_schema($this->schema);
        }
        
        // Define JSON Schema Draft 4 compliant schema
        $this->schema = array(
            '$schema'    => 'http://json-schema.org/draft-04/schema#', // Schema version identifier
            'title'      => 'resource',     // Human-readable schema name
            'type'       => 'object',       // Root type must be 'object' for REST resources
            'properties' => array(          // Define all allowed properties
                'name' => array(
                    'type'      => 'string',    // Data type for validation
                    'required'  => true,        // Makes field mandatory for creation
                    'minLength' => 1,           // Prevents empty strings
                    'maxLength' => 100,         // Prevents oversized input
                ),
                'email' => array(
                    'type'     => 'string',     // Base type validation
                    'format'   => 'email',      // WordPress validates email format
                    'required' => true,         // Mandatory field
                ),
            ),
        );
        
        // add_additional_fields_schema() is called again to ensure any custom fields
        // registered after schema creation are included in the final schema
        return $this->add_additional_fields_schema($this->schema);
    }
    
    public function create_item_permissions_check($request) {
        return current_user_can('edit_posts');
    }
    
    public function create_item($request) {
        // At this point, validation and sanitization have already occurred automatically
        // because we used get_endpoint_args_for_item_schema() in register_routes()
        // All request parameters are now validated against schema and sanitized
        
        // prepare_item_for_database() should transform sanitized request data
        // into the format needed for storage (e.g., database structure)
        $item = $this->prepare_item_for_database($request);
        
        // Always check if preparation resulted in an error
        if (is_wp_error($item)) {
            return $item;
        }
        
        // Save to database and return standardized REST response
        // rest_ensure_response() converts various return types to WP_REST_Response
        return rest_ensure_response($item);
    }
}
```

## Schema Definition Rules

**ALWAYS specify 'type'** - Missing type triggers WordPress warnings.

**Use these validation properties by type:**

```php
// String validation
'field' => array(
    'type'      => 'string',
    'minLength' => 1,
    'maxLength' => 255,
    'pattern'   => '^[A-Za-z0-9]+$',
    'format'    => 'email', // email, date-time, uri, ip
    'enum'      => array('value1', 'value2'),
)

// Number validation
'field' => array(
    'type'       => 'number', // or 'integer'
    'minimum'    => 0,
    'maximum'    => 100,
    'multipleOf' => 0.01,
)

// Array validation
'field' => array(
    'type'        => 'array',
    'items'       => array('type' => 'string'),
    'minItems'    => 1,
    'maxItems'    => 10,
    'uniqueItems' => true,
)

// Object validation
'field' => array(
    'type'       => 'object',
    'properties' => array(
        'name' => array('type' => 'string'),
    ),
    'required'             => array('name'),
    'additionalProperties' => false,
)
```

## Error Handling Pattern

```php
public function create_item($request) {
    // Custom validation example
    $email = $request->get_param('email');
    
    // First validate against schema
    $valid = rest_validate_value_from_schema(
        $email, 
        array('type' => 'string', 'format' => 'email'), 
        'email'
    );
    
    if (is_wp_error($valid)) {
        return $valid;
    }
    
    // Then sanitize
    $clean_email = rest_sanitize_value_from_schema(
        $email, 
        array('type' => 'string', 'format' => 'email'), 
        'email'
    );
    
    if (is_wp_error($clean_email)) {
        return $clean_email;
    }
    
    // Business logic validation
    if (email_exists($clean_email)) {
        return new WP_Error(
            'rest_email_exists',
            'Email already registered.',
            array('status' => 409)
        );
    }
    
    // Process request...
}
```

## Common Mistakes to Avoid

❌ **NEVER do this:**
```php
// Missing validation callbacks
'args' => array(
    'field' => array('type' => 'string'), // No validation!
)

// Manual registration without WP_REST_Controller
register_rest_route('namespace', '/endpoint', array(
    'callback' => 'function_name', // Avoid this pattern
));
```

✅ **ALWAYS create a class that extends `WP_REST_Controller`**

## Security Checklist

- [ ] Extends WP_REST_Controller
- [ ] Uses `get_endpoint_args_for_item_schema()` for automatic validation
- [ ] All schema properties include 'type'
- [ ] Permission callbacks check capabilities
- [ ] Custom validation uses both schema functions in order
- [ ] Error messages don't expose sensitive data

## Registration Pattern

```php
add_action('rest_api_init', function() {
    $controller = new My_Controller();
    $controller->register_routes();
});
```

**Remember**: WP_REST_Controller automatically handles validation and sanitization when using `get_endpoint_args_for_item_schema()`. Manual callback specification is only needed for custom validation logic beyond schema constraints.
