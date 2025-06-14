# WordPress Query Guidelines
When instructed to create or revise queries, use the following best practices. When generating code add comments to explain why a query was created the way it was.
- Prefer using WP_Query over get_posts() unless the user specifically asks for the latter.
- Do not re-set default arguments for the WP_Query

## posts_per_page
   The posts_per_page parameter directly controls the number of posts returned by the query. If you only need one post, set posts_per_page to 1 to reduce unnecessary load.
```php
$query = new WP_Query([
    'post_type'      => 'post',
    'posts_per_page' => 1,
]);
 ```
- Use 1 if you only need a single post.
- Keep result sets under 100 posts unless paginating or processing in batches.
- Avoid -1 (to get all posts), as this can quickly consume memory on larger sites.

## no_found_rows
It is needed when you’re paginating results but without, you can disable this calculation to improve performance.
```php
$query = new WP_Query([
    'post_type'      => 'post',
    'posts_per_page' => 10,
    'no_found_rows'  => true,
]);
 ```
Use when: You don’t need pagination, such as on homepage widgets or API responses.

## ignore_sticky_posts
If sticky posts aren’t important for your use case, you can tell WordPress to ignore them.
```php
$query = new WP_Query([
    'post_type'           => 'post',
    'posts_per_page'      => 10,
    'ignore_sticky_posts' => true,
]);
 ```
Use when: You don’t need sticky posts interfering with your query, such as in custom feeds or filtered post lists.

## Controlling returned post fields
The fields parameter controls which parts of the WP_Post object are returned. If you only need specific data (e.g., just the post IDs), setting this to ‘ids’ can reduce memory usage and improve performance.
```php
$query = new WP_Query([
    'post_type' => 'post',
    'fields'    => 'ids',
]);
 ```
Options:
- `all` – Returns full post objects (default).
- `ids` – Returns an array of post IDs.
- `id=>parent` – Returns an associative array of IDs and their parent IDs.

Use `ids` or `id=>parent` when you don’t need the full post object, such as for bulk processing.

## Faster Taxonomy Queries with term_taxonomy_id
If you’re querying posts by taxonomy, you can optimise performance by querying by term_taxonomy_id directly. This avoids unnecessary extra SQL joins and lookups in the terms and term_taxonomy tables and can significantly speed up the query.
```php
$query = new WP_Query([
    'tax_query' => [
        [
            'taxonomy'         => 'category',
            'field'            => 'term_taxonomy_id',
            'terms'            => [23, 47],
            'include_children' => false,
        ],
    ],
]);
 ```

## Optimising Queries with post__in
When using the post__in parameter to query a specific set of posts by their post IDs, you can improve performance by setting the posts_per_page parameter to match the count of the post IDs array. This ensures the query doesn’t needlessly paginate or fetch extra posts.
```php
$post_ids = [1, 2, 3, 4, 5]; // Array of post IDs

$query = new WP_Query([
    'post_type'      => 'post',
    'post__in'       => $post_ids,
    'posts_per_page' => count( $post_ids ), // Match posts_per_page with array length
]);
 ```

## Use get_post() Instead of WP_Query for Single Posts
If you already have the post ID and only need to retrieve one post, you can bypass WP_Query entirely and use the simpler and more efficient get_post() function.
```php
$post = get_post( 123 ); // Retrieve post by ID
 ```
`get_post()` is much faster than WP_Query when you’re only querying one post by its ID.

## Preloading meta-cache, term-cache and menu-item-cache
Control whether WordPress preloads metadata, taxonomy terms, and menu item data.
```php
$query = new WP_Query([
    'post_type'               => 'post',
    'posts_per_page'          => 10,
    'update_post_meta_cache'  => false,
    'update_post_term_cache'  => false,
    'update_menu_item_cache'  => false,
]);
 ```
Use when: You’re only displaying basic post data (like IDs or titles) or intend to fetch metadata or terms later.

Important: update_menu_item_cache should only be enabled when querying nav_menu_item post types. For other post types, leave it disabled to avoid unnecessary overhead.

## WPML Support
When working with WPML multilingual plugin we need to suppress the filters to get posts in all languages. If we do not add the argument posts are only returned in the current language. By default WP_Query returns only posts in the current language.
```php
$query = new WP_Query([
    'post_type'         => 'post',
    'suppress_filters'  => true, // Required to fetch posts in all languages with WPML
]);
 ```

## `*_in` / `*_not_in` Essentials
- An empty array (`[]`) in `post__in`, `category__in`, etc., drops the filter and returns **all** matching records. Always test the array before attaching the argument.
- Skip the `*_not_in` parameter altogether if you have nothing to exclude.
- To show zero results: Force explicitly with `post__in => [0]` and `posts_per_page => 0`.


```php
$post_ids = get_selected_ids() ?: [0]; // may be empty, if that is the case explicitly set to [0] to return zero results

$args = [ 'post_type' => 'post' ];
$args['post__in']       = $post_ids;
$args['posts_per_page'] = count( $post_ids );

$query = new WP_Query( $args );
```