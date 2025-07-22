# WordPress PHPStan Configuration & Rules

Static analysis configuration and patterns for WordPress development with PHPStan. ALWAYS follow these when running phpstan.

## Core Configuration Requirements

### Baseline PHPStan Configuration
Always include WordPress-specific extensions and rules in `phpstan.neon`:

```neon
includes:
    - phpstan-baseline.neon
parameters:
    level: 5
    paths:
        - src/
        - includes/
    excludePaths:
        - vendor/
        - vendor-prefixed/
        - node_modules/
        - tests/
    ignoreErrors:
        # WordPress core compatibility patterns (add as needed)
```

## Third-Party Class Handling

### Plugin/Theme Compatibility Patterns
When PHPStan reports "Class not found" or "unknown class" errors for legitimate third-party dependencies, add specific ignore patterns:

```neon
ignoreErrors:
    # Admin Columns Pro
    - '#.*(unknown class|invalid type|call to method .* on an unknown class) AC\\ListScreen.*#'
    
    # WooCommerce
    - '#.*(unknown class|invalid type|call to method .* on an unknown class) WC_.*#'
    
    # Advanced Custom Fields
    - '#.*(unknown class|invalid type|call to method .* on an unknown class) ACF_.*#'
    
    # Elementor
    - '#.*(unknown class|invalid type|call to method .* on an unknown class) Elementor\\.*#'
    
    # Yoast SEO
    - '#.*(unknown class|invalid type|call to method .* on an unknown class) WPSEO_.*#'
```

**Pattern Creation Rules:**
- Cover all error variations: `unknown class`, `invalid type`, `call to method .* on an unknown class`
- Be specific enough to target only intended classes
- Include descriptive comments explaining which plugin/theme
- Group related patterns for the same plugin together

**When to Add Exceptions:**
- Only for legitimate third-party dependencies that your code integrates with
- Document each pattern with source plugin/theme comment
- Verify the plugin is actually installed/required in your environment

## WordPress REST API Type Annotations

### Request Parameter Typing
PHPStan cannot infer valid request parameters from REST API schemas. Always provide explicit type hints:

```php
/**
 * Handle REST API request.
 *
 * @param WP_REST_Request $request Full details about the request.
 * @return WP_REST_Response|WP_Error Response object on success, error on failure.
 *
 * @phpstan-param WP_REST_Request<array{
 *     post?: int, 
 *     orderby?: string, 
 *     meta_key?: string,
 *     per_page?: int,
 *     status?: array<string>
 * }> $request
 */
public function get_items($request) {
    $post_id = $request->get_param('post');
    // PHPStan now knows $post_id is int|null
}
```

### Schema-Based Type Definitions
For complex schemas, define reusable types:

```php
/**
 * @phpstan-type PostRequestParams array{
 *     title?: string,
 *     content?: string,
 *     status?: 'publish'|'draft'|'private',
 *     meta?: array<string, mixed>
 * }
 * 
 * @phpstan-param WP_REST_Request<PostRequestParams> $request
 */
```

## WordPress Core Integration

### Global Variables & Hook Callbacks
```php
/**
 * @global wpdb $wpdb
 * @global WP_Post $post
 */
function process_post_data(): void {
    global $wpdb, $post;
    $results = $wpdb->get_results($wpdb->prepare("SELECT * FROM {$wpdb->posts} WHERE post_parent = %d", $post->ID));
}

/**
 * @param string $new_status
 * @param string $old_status 
 * @param WP_Post $post
 */
function handle_transition(string $new_status, string $old_status, WP_Post $post): void { /* ... */ }
add_action('transition_post_status', 'handle_transition', 10, 3);
```

### Database & Iterables
```php
/**
 * @return array<WP_Post> WP_Post objects.
 */
function get_custom_posts(): array {
    $posts = get_posts(['post_type' => 'custom_type', 'numberposts' => -1]);
    return $posts; // PHPStan knows this returns WP_Post[]
}

/**
 * @return array<object{id: int, name: string}> Database results.
 */
function get_user_data(): array {
    global $wpdb;
    $results = $wpdb->get_results("SELECT id, name FROM users", OBJECT);
    return $results ?: [];
}
```

### Hooks (Actions & Filters)
Doclbocks of apply_filters() and do_action() are validated. The type of the first @param is definitive. No furhter validation of the filter output is needed. Considering strong stypings, an error as the result of a type missmatch caused by a faulty implementation of a third party using the hook is desired and does not need to be handled.

```php
/**
 * Allows hooking into formatting of the price
 *
 * @param string $$value The formatted price.
 * @param string|null $formatted The formatted price.
 * @param float $price The raw price.
 * @param string $locale Locale to localize pricing display.
 * @param string $currency Currency Symbol.
 */
return apply_filters( 'autoscout_vehicle_price_formatted', $formatted, $price, $locale, $currency );
```

## Action Scheduler
```php
/**
 * @param array{user_id: int, email: string, data: array<string, mixed>} $args
 */
function process_scheduled_email(array $args): void {
    $user_id = $args['user_id'];  // PHPStan knows this is int
    // Process the scheduled task
}

as_schedule_single_action(time() + 3600, 'process_scheduled_email', [
    'user_id' => 123, 'email' => 'user@example.com', 'data' => ['key' => 'value']
]);
```

## Baseline Management

### Baseline Commands
```bash
# Generate baseline
vendor/bin/phpstan analyse --generate-baseline phpstan-baseline.neon

# Update baseline
vendor/bin/phpstan analyse --generate-baseline
```

**Best Practices:** Don't commit new errors to baseline; review baseline changes in PRs; gradually reduce errors over time.
