# WordPress Code Development

## Environment & Folder Structure

- **Development Context**: Plugins and themes are located within a full WordPress environment, typically under `wp-content/plugins` or `wp-content/themes`.

## General Coding & Language Guidelines

### All Languages

- **Adhere to WordPress code styling**
- **Provide docblocks** for all functions, classes, and files
- **English only**: Always generate code and comments in English, regardless of input language
- **Early Return/Exit**: Use early return/exit patterns for condition checks to simplify nesting and enhance readability
- **Focus on readability & clean code principles**

### PHP Specific Guidelines

- Ensure code is **PHPCS compatible**
- Use **Yoda Conditions** for all conditional expressions
- Default to a **minimum PHP compatibility of 8.0** (unless another version is explicitly requested)
- **Strong typings**: Use PHPStan doc block notation (especially for array type hints)
- **Object-Oriented Approach**:  
  Write OOP code by default unless the user asks for procedural/non-OOP structure (e.g., `functions.php`)
- **File Operations**:  
  Use `WP_Filesystem` APIs instead of PHP native file functions whenever possible
- **Error Handling & Exception Logging**:
    - In non-production: Use `wp_trigger_error( $function_name, $message, $error_level )` at the appropriate level
    - In specific contexts (e.g., Action Scheduler in production), direct Exception throwing may be required—do **not** use `wp_trigger_error` in those cases
- Add **translator comments** for internationalization, e.g.:  
  `/* translators: 1: WordPress version number, 2: plural number of bugs. */`
- When working with times use DateTime or preferably DateTimeImmutable instances.
    - `current_datetime()` current time as DateTimeImmutable
    - `get_post_datetime()` post time as DateTimeImmutable
    - `get_post_timestamp()` post time as Unix timestamp.
    - `wp_date( string $format, int $timestamp = null, DateTimeZone $timezone = null ): string|false`. Timestamp is the curent time by default. Timezone defaults to timezone from site settings.

### JavaScript Guidelines

- **Framework Selection**:  
  Provide code for framework per user requirement: `"Vanilla Javascript"`, `"ReactJs"`, `"AlpineJs"`, or `"WordPress Interactivity API"`. Default to "Vanilla Javascript" if unspecified.
- **JSDoc Docblocks**:  
  Every function/class must use JSDoc doc blocks, fully describing parameters, errors, and return values in full sentences
- **ESLint Comments**:  
  Only allow `// eslint-disable-line` comments for: `camelcase`, `console.log`, and `console.error`
- **AlpineJs Pattern**:  
  Keep AlpineJS `data` in its **own file** and import as:
  ```js
  import Alpine from 'alpinejs'
  import dropdown from './dropdown.js'
  Alpine.data('dropdown', dropdown)
  ```
  In context of the plugin boilerplate alpine components should be added to resources/admin/js or resources/frontend/js. Each component should be its own file with one default function like
    ```js
    export default () => ({
        open: false,
     
        toggle() {
            this.open = ! this.open
        }
    })
    ```

## Debugging & Quality Assurance

### Static code analysis (PHPStan)
#### Third party classes
When PHPStan reports "Class not found" or "unknown class" errors, add specific ignore patterns to the ignoreErrors section in phpstan.neon:
```neon
ignoreErrors:
    # Ignore specific third-party class errors with descriptive comments
    - '#.*(unknown class|invalid type|call to method .* on an unknown class) AC\\ListScreen.*#' # Admin Columns Pro classes
    - '#.*(unknown class|invalid type|call to method .* on an unknown class) ExtendedPlugin\\ClassName.*#' # Other plugin example
```
**Create patterns that:**
- Cover all error variations (unknown class, invalid type, method calls)
- Are specific enough to target only intended classes
- Include comments explaining which plugin/theme the ignored class belongs to

**When to add exceptions:**
- Only ignore errors for legitimate third-party dependencies
- Document each pattern with a comment indicating the source plugin/theme
- Group related patterns for the same plugin together

#### WordPress Rest API Typings
PHPStan cannot know the valid request parameters defined in your schema. Therefore, we always have to let PHPStan know which parameters are available. Like this

```php
/**
 * @param WP_REST_Request $request Full details about the request.
 * @return WP_REST_Response|WP_Error Response object on success, WP_Error object on failure.
 *
 * @phpstan-param WP_REST_Request<array{post?: int, orderby?: string}> $request
 */
```