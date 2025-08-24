# WordPress REST API – Validation & Sanitization
## Purpose

Make request handling **predictable, secure, and self-documenting** by using **JSON Schema everywhere**:

* **Every resource controller** must publish a canonical item schema via `get_item_schema()`.
* **Every endpoint** must validate & sanitize **all inputs** using a JSON Schema (either derived from the item schema or declared inline for custom actions).

---

## Golden Rules

1. **One resource ⇒ one controller ⇒ one canonical schema.**
   Implement `get_item_schema()` in each resource controller and attach it to the route(s) via `'schema' => [ $this, 'get_public_item_schema' ]`.

2. **Never accept input without a schema.**
   For write endpoints, derive args from the item schema with `get_endpoint_args_for_item_schema()` **or** provide an inline schema for custom actions. Do **not** rely on regex-only URL params or omit schema keywords.

3. **Make required fields explicit for POST/PUT.**
   `get_endpoint_args_for_item_schema()` sets `required => false` by default—override per method if you need hard requirements.

4. **Use contexts in your response.**
   Drive `view`/`edit`/`embed` visibility via the schema (`context`) and `filter_response_by_context()` inside `prepare_item_for_response()`.

---

## When to use `get_item_schema()` (Per Resource)

Use `get_item_schema()` to define the **shape of your resource** (e.g., Project, Order, Invoice). This schema is the **single source of truth** for:

* **Discovery:** exposed via `OPTIONS` and `'schema' => [ $this, 'get_public_item_schema' ]` on routes that return the resource.
* **Response structure & contexts:** which fields exist, their types, and in which `context` they are exposed.
* **Generating write-args:** feed it into `get_endpoint_args_for_item_schema()` for POST/PUT/PATCH to obtain validated/sanitized input args.

> Use `get_item_schema()` for all **CRUD routes and any route that returns the resource** (or a collection of it). Do **not** redefine the same fields as ad‑hoc args schemas across endpoints.

### Minimal controller pattern

```php
class Projects_Controller extends WP_REST_Controller {
    protected $namespace = 'my-plugin/v1';
    protected $rest_base = 'projects';
    protected $schema = null;

    public function register_routes() {
        // Collection
        register_rest_route($this->namespace, '/' . $this->rest_base, [
            [
                'methods'  => WP_REST_Server::READABLE,
                'callback' => [ $this, 'get_items' ],
                'permission_callback' => [ $this, 'get_items_permissions_check' ],
                'args' => $this->get_collection_params(), // may include schema-like filters
            ],
            'schema' => [ $this, 'get_public_item_schema' ],
        ]);

        // Single item
        register_rest_route($this->namespace, '/' . $this->rest_base . '/(?P<id>\d+)', [
            'args' => [ 'id' => [ 'type' => 'integer', 'minimum' => 1 ] ],
            [
                'methods'  => WP_REST_Server::READABLE,
                'callback' => [ $this, 'get_item' ],
                'permission_callback' => [ $this, 'get_item_permissions_check' ],
            ],
            [
                'methods'  => WP_REST_Server::EDITABLE, // PUT/PATCH
                'callback' => [ $this, 'update_item' ],
                'permission_callback' => [ $this, 'update_item_permissions_check' ],
                'args'     => (function () {
                    $args = $this->get_endpoint_args_for_item_schema(WP_REST_Server::EDITABLE);
                    // Enforce required fields for updates if needed
                    // $args['name']['required'] = true;
                    return $args;
                })(),
            ],
            [
                'methods'  => WP_REST_Server::DELETABLE,
                'callback' => [ $this, 'delete_item' ],
                'permission_callback' => [ $this, 'delete_item_permissions_check' ],
            ],
            'schema' => [ $this, 'get_public_item_schema' ],
        ]);
    }

    public function get_item_schema() {
        if ($this->schema) {
            return $this->add_additional_fields_schema($this->schema);
        }
        $this->schema = [
            '$schema'    => 'http://json-schema.org/draft-04/schema#',
            'title'      => 'project',
            'type'       => 'object',
            'properties' => [
                'id'    => [ 'type' => 'integer', 'readonly' => true ],
                'name'  => [ 'type' => 'string', 'minLength' => 1, 'maxLength' => 200 ],
                'status'=> [ 'type' => 'string', 'enum' => ['draft','published'], 'default' => 'draft' ],
                'meta'  => [
                    'type'                 => 'object',
                    'additionalProperties' => [ 'type' => 'string' ],
                    'default'              => (object) [],
                ],
            ],
            // 'required' => ['name'], // use object-level required if you need it for POST; override per method
        ];
        return $this->add_additional_fields_schema($this->schema);
    }
}
```

---

## When to **manually** register schema in `args`

Use inline `args` schemas when the endpoint is **not** expressing the full resource representation, e.g. **custom actions** or **non-CRUD** routes:

* **Actions on a resource** (e.g., `/projects/{id}/publish`, `/projects/{id}/contextualize`).
* **Bulk/Batch** operations with special payloads.
* **Search/filter endpoints** that accept complex query parameters not covered by the item schema.

In these cases, define a **complete JSON Schema** for each request parameter under `args`. WordPress will **automatically** validate & sanitize values against that schema. Avoid raw params without schema or callbacks.

---

## How the item schema is used across operations

* **Create (POST)** and **Update (PUT/PATCH):**
  Use `get_endpoint_args_for_item_schema(\WP_REST_Server::CREATABLE|EDITABLE)` to derive per-method input args **from the item schema**. Override `required` flags per method if necessary.

* **Read (GET):**
  Use `get_public_item_schema()` on routes that return the resource so clients can discover fields & contexts. For collection filters, define schema-like `args` (type, enum, items, etc.) on the route.

* **Response shaping:**
  In `prepare_item_for_response()`, populate fields in accordance with the item schema and call `filter_response_by_context()` so only fields allowed for the given `context` are emitted.

> Note: Core does **not** auto-validate responses against the schema—ensure your `prepare_item_for_response()` matches the schema.

---

## Example: Custom action with complex array input

You have an action to contextualize a Project with an **array of link objects**.

### Preferred (automatic) – pass the schema directly as the arg

```php
$links_schema = [
    '$schema'    => 'http://json-schema.org/draft-04/schema#',
    'title'      => 'contextualize_links',
    'type'       => 'array',
    'description'=> 'Array of links with URLs and optional labels',
    'items'      => [
        'type'       => 'object',
        'properties' => [
            'url'   => [ 'type' => 'string', 'format' => 'uri', 'minLength' => 1, 'maxLength' => 2000 ],
            'label' => [ 'type' => 'string', 'minLength' => 0, 'maxLength' => 255 ],
        ],
        'required'             => [ 'url' ],
        'additionalProperties' => false,
    ],
    'minItems'    => 0,
    'maxItems'    => 100,
    'uniqueItems' => false,
];

register_rest_route(
    'sw-video-processing/v1',
    '/project/(?P<id>\d+)/contextualize',
    [
        [
            'methods'             => WP_REST_Server::CREATABLE,
            'callback'            => [ $this, 'contextualize' ],
            'permission_callback' => [ Auth::class, 'app_to_app_auth' ],
            'args'                => [
                'links' => array_merge( $links_schema, [ 'required' => false ] ),
            ],
        ],
    ]
);
```

*Outcome:* WP will **validate** via `rest_validate_value_from_schema()` and **sanitize** via `rest_sanitize_value_from_schema()` automatically; `$request->get_param('links')` is already normalized.
