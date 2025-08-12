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
For comprehensive PHPStan configuration, third-party class handling, REST API typings, and WordPress-specific patterns, see `rules/quality-assurance/phpstan.md`.

## Documentation & Repository Files

### README files

| File             | Purpose & Audience                                                                                                                      | Required Format                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Governance                                                                                                                         |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **`readme.txt`** | Mandatory for assets published via the WordPress .org Plugin Directory. Parsed by wp.org to generate the plugin listing and change-log. | *Plain-text* with the canonical WordPress readme header and section order:<br>`=== Plugin Name ===`<br>`Contributors:` …<br>`Tags:` …<br>`Requires at least:` …<br>`Tested up to:` …<br>`Requires PHP:` …<br>`Stable tag:` …<br><br>Followed by the standard headings (in this order):<br>`== Description ==`<br>`== Installation ==`<br>`== Screenshots ==`<br>`== Frequently Asked Questions ==`<br>`== Changelog ==`<br>`== Upgrade Notice ==`<br><br>- Use limited wp.org markup (`*italic*`, `**bold**`, back-ticked code).<br>- No tables, HTML, or GitHub-only Markdown extensions.<br>- Keep the first 150 characters of *Description* marketing-focused: that snippet is shown in search results. | **Must** follow these rules. Treated as the single source of truth for plugin meta-data across all distribution channels.          |
| **`README.md`**  | Optional, aimed at developers browsing the repository (e.g., on GitHub, GitLab, Bitbucket).                                             | Any valid Markdown. Badges, tables, images, extended syntax all allowed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Its structure, tone, and tooling badges are left entirely to the repository maintainers. |

#### Practical Differences (TL;DR)

1. **Consumer**: `readme.txt` is machine-read by WordPress.org; `README.md` is human-read by developers.
2. **Syntax**: `readme.txt` → strict wp.org subset; `README.md` → full Markdown.
3. **Metadata**: Only `readme.txt` controls plugin header fields, screenshots, and changelog on wp.org.
4. **Governance**: You **must** comply with the `readme.txt` template; you’re free to do whatever you like with `README.md`.


## Context Retrieval
Use the context7 mcp server if it is available to lookup examples or documentation to get more details about tools, plugins and software. The following libraries are out of interest. Call them directly with their id:
* Bricks Builder (Page Builder for WordPress): `/digisavvy-inc/bricks-builder-docs`
* Advanced Custom Field Pro ACF (Custom Fields, Post Types, Taxonomies): `/advancedcustomfields/acf`
* WP Gridbuilder (Filtering, Facets): `/context7/wpgridbuilder`
* WS Form (Forms that are highly dynamic and can be customized using code): `/context7/wsform-knowledgebase`
