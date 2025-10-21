# Documentation & Repository Files

Guidline for how to document code functionality, changelogs and developer instructions.

## README files

| File             | Purpose & Audience                                                                                                                      | Required Format                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Governance                                                                                                                         |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| [**`readme.txt`**](https://developer.wordpress.org/plugins/wordpress-org/how-your-readme-txt-works/)| Mandatory for assets published via the WordPress .org Plugin Directory. Parsed by wp.org to generate the plugin listing and change-log. | *Plain-text* with the canonical WordPress readme header and section order:<br>`=== Plugin Name ===`<br>`Contributors:` …<br>`Tags:` …<br>`Requires at least:` …<br>`Tested up to:` …<br>`Requires PHP:` …<br>`Stable tag:` …<br><br>Followed by the standard headings (in this order):<br>`== Description ==`<br>`== Installation ==`<br>`== Screenshots ==`<br>`== Frequently Asked Questions ==`<br>`== Changelog ==`<br>`== Upgrade Notice ==`<br><br>- Use limited wp.org markup (`*italic*`, `**bold**`, back-ticked code).<br>- No tables, HTML, or GitHub-only Markdown extensions.<br>- Keep the first 150 characters of *Description* marketing-focused: that snippet is shown in search results. | **Must** follow these rules. Treated as the single source of truth for plugin meta-data across all distribution channels.          |
| **`README.md`**  | Optional, aimed at developers browsing the repository (e.g., on GitHub, GitLab, Bitbucket).                                             | Any valid Markdown. Badges, tables, images, extended syntax all allowed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Its structure, tone, and tooling badges are left entirely to the repository maintainers. |

## Docs folder
It is common practice to add a docs folder in the root of a repository containing markdown files. Follow these rules when adding documentation:

* Create well foratted markdown with a hierarchical structure of headings
* Add examples but keep them as short as possible to demonstrate a certain implementation.
* Be precise and not overly verbose while keeping the docs human and machine readable

### Docs folder structure
The following folder structure is a good starter.
```
docs/
├── README.md                     # Introduction to docs/plugin/theme functionality
├── architecture/
|   ├── Readme.md
|   └── Detail1.md
└── guides/
    ├── Howto1.md
    └── Howto2.md
```
