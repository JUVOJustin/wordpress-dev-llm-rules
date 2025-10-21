# WordPress Code Development
Base Coding rules for WordPress. ALWAYS read this when working on a WordPress project

## General Coding & Language Guidelines

### All Languages

- **Adhere to WordPress code styling**
- **Provide docblocks** for all functions, classes, and files
- **English only**: Always generate code and comments in English, regardless of input language
- **Early Return/Exit**: ALWAYS use early return/exit patterns for condition checks to simplify nesting and enhance readability
- **Focus on readability & clean code principles**

### PHP Specific Guidelines

- Ensure code is **PHPCS compatible**
- Use **Yoda Conditions** for all conditional expressions
- Default to a **minimum PHP compatibility of 8.0** (unless another version is explicitly requested)
- **Strong typings**: Use PHPStan doc block notation (especially for array type hints)
- **Object-Oriented Approach**: Write OOP code by default unless the user asks for procedural/non-OOP structure (e.g., `functions.php`)
- **File Operations**: Use `WP_Filesystem` APIs instead of PHP native file functions whenever possible
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

#### Naming
ALWAYS adhere to the naming conventions outlined in this example:
```php
function some_name( $some_variable ) {}
class Walker_Category extends Walker {}
class WP_HTTP {}
interface Mailer_Interface {}
trait Forbid_Dynamic_Properties {}
enum Post_Status {}
```

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
For comprehensive PHPStan configuration, third-party class handling, REST API typings, and WordPress-specific patterns, see `rules/quality-assurance/phpstan.md`.

#### Practical Differences (TL;DR)

1. **Consumer**: `readme.txt` is machine-read by WordPress.org; `README.md` is human-read by developers.
2. **Syntax**: `readme.txt` → strict wp.org subset; `README.md` → full Markdown.
3. **Metadata**: Only `readme.txt` controls plugin header fields, screenshots, and changelog on wp.org.
4. **Governance**: You **must** comply with the `readme.txt` template; you’re free to do whatever you like with `README.md`.


## Context Retrieval
Use mcp tool if available to lookup examples or documentation to get more details about tools, plugins and software. The following libraries are of interest. Call them directly with their id:

|ID|Name|Description|MCP Server
|---|---|---|---|
|/digisavvy-inc/bricks-builder-docs|Bricks Builder|Agency focused Page/Theme Builder|context7|
|/advancedcustomfields/acf|Advanced Custom Field Pro (ACF)|Adding Custom Fields, Post Types, Taxonomies|context7|
|/context7/wpgridbuilder|WP Gridbuilder|Advanced filtering of queries using Facets|context7|
|/context7/wsform-knowledgebase|WS Form|Forms that are highly dynamic and can be customized using code|context7|
|/context7/automaticcss|Automatic.css (ACSS)|Modern css framework that closely works with etch builder, bricks builder or standalone|context7|
|/wordpress/abilities-api|Abilities API|Declaring and discovering abilities in a standardized way. Can expose Abilities to AI.|context7|
|/wordpress/mcp-adapter|MCP Adapter|MCP integration that allows developers to expose abilities, registered with abilities API, as Model Context Protocol (MCP) tools, resources, and prompts for AI agents.|context7|
