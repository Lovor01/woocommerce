# Push Notifications - Send Notification Implementation Plan

## Overview

Implement async push notification sending when orders are created or reviews are posted. Notifications are sent via WordPress.com's public API.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Original Request                                                        │
│                                                                         │
│  Hook fires (order status change / comment_post)                       │
│       │                                                                 │
│       ▼                                                                 │
│  PushNotificationTrigger checks conditions                             │
│       │                                                                 │
│       ▼                                                                 │
│  Atomic meta write to claim notification (add_post_meta unique=true)   │
│       │                                                                 │
│       ├── Failed (already exists) → Exit                               │
│       │                                                                 │
│       ▼                                                                 │
│  Queues notification data for shutdown                                 │
│       │                                                                 │
│       ▼                                                                 │
│  On shutdown: spawn async request (non-blocking, HMAC token in header) │
│       │                                                                 │
│  Response sent to user ◄───────────────────────────────────────────────┤
└───────│─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Spawned Async Request (separate PHP process)                            │
│                                                                         │
│  PushNotificationRestController receives request                       │
│       │                                                                 │
│       ▼                                                                 │
│  Verifies HMAC token signature (stateless, no DB lookup)               │
│       │                                                                 │
│       ▼                                                                 │
│  Retrieves device tokens for admin/shop_manager users                  │
│       │                                                                 │
│       ▼                                                                 │
│  PushNotificationSender sends to WordPress.com API (blocking)          │
│  (includes idempotency_key for WPCOM-side deduplication)               │
│       │                                                                 │
│       ├── Success → Done                                               │
│       │                                                                 │
│       └── Failure → PushNotificationJobManager schedules retry         │
│                          │                                              │
│                          ▼                                              │
│                     ActionScheduler                                     │
│                          │                                              │
│                          ▼                                              │
│                     PushNotificationRetryProcessor                      │
│                     (up to 5 retries: 10s, 60s, 5m, 15m, 60m)          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Handling Database Replication Lag

In multi-server environments with read replicas, replication lag can cause issues:
- Server A writes data, Server B can't read it yet

**Our mitigations:**

1. **HMAC-signed tokens** - Authenticates that the request originated from our own server.
   Token contains `{type, resource_id, expires}` signed with `wp_salt('auth')`.
   This follows the same pattern as `StoreApi/Utilities/JsonWebToken.php` which uses
   HMAC-SHA256 with WordPress salts for stateless authentication. External attackers
   cannot forge valid tokens without access to the server's secret salts.

2. **Meta flag check** - Similar to stock reduction pattern in `wc-stock-functions.php`.
   Check if meta exists, then set it if not. This is NOT truly atomic (small race window),
   but the consequence is minor (extra HTTP request) and WPCOM deduplicates via idempotency key.

3. **Idempotency key** - Each notification includes a unique key (`{type}_{resource_id}`).
   WPCOM can deduplicate if the same notification arrives twice.

## Deduplication Strategy

Three layers prevent duplicate notifications:

| Layer | Mechanism | What It Prevents |
|-------|-----------|------------------|
| **Trigger** | Meta flag on order/comment | Same resource triggering multiple times (e.g., repeated status changes) |
| **Retry** | None (intentional) | N/A - retries should attempt to send |
| **WPCOM** | `idempotency_key` | Same notification delivered to device twice |

**Why retries don't check meta:**
- Meta is set at trigger time to *claim* the notification
- If a retry is scheduled, the previous send *failed*
- We want retries to attempt sending
- WPCOM's idempotency_key handles edge cases where notification somehow gets delivered twice

## File Structure

```
src/Internal/PushNotifications/
├── PushNotifications.php                         # Main class (existing)
├── PushNotificationLoopbackSpawner.php           # NEW: Spawns async process via loopback
├── PLAN.md                                       # This file
├── Controllers/
│   ├── PushTokenRestController.php               # Existing: Token management
│   └── PushNotificationRestController.php        # NEW: Async endpoint for sending
├── Notifications/                                # NEW: Notification type classes
│   ├── AbstractPushNotification.php              # Base class with common logic
│   ├── NewOrderPushNotification.php              # Order notification (title, message, icon)
│   └── NewReviewPushNotification.php             # Review notification (title, message, icon)
├── Triggers/                                     # NEW: Hook handlers (run in main request)
│   ├── AbstractPushNotificationTrigger.php       # Base trigger class
│   ├── NewOrderPushNotificationTrigger.php       # Order notification trigger
│   └── NewReviewPushNotificationTrigger.php      # Review notification trigger
├── AsyncTasks/                                   # NEW: Run in async context
│   ├── PushNotificationSender.php                # Sends to WordPress.com API
│   ├── PushNotificationJobManager.php            # ActionScheduler job management
│   └── PushNotificationRetryProcessor.php        # Processes all retries (loopback + WPCOM failures)
├── DataStores/
│   └── PushTokensDataStore.php                   # Existing
├── Entities/
│   └── PushToken.php                             # Existing
└── Exceptions/
    └── PushTokenNotFoundException.php            # Existing
```

## Implementation Details

### 0. PushToken Entity Updates

Add a `get_environment()` method to the existing `PushToken` entity to derive the WPCOM environment from the token's origin:

```php
// In PushToken.php - add this method:

/**
 * Get the WPCOM push notification environment for this token.
 *
 * Origins ending in ':dev' are sandbox environments (used for development builds).
 * All other origins are production.
 *
 * @return string 'sandbox' or 'production'
 *
 * @since 10.5.0
 */
public function get_environment(): string {
    return str_ends_with($this->origin ?? '', ':dev') ? 'sandbox' : 'production';
}
```

### 1. Notification Classes (One Per Type)

Each notification type has its own class that encapsulates title, message, icon, and metadata.

**Why use a private constructor with a static `create()` method?**
- Constructors cannot return `null`, but we need to gracefully handle cases where the resource doesn't exist
- The `create()` method validates the resource exists and returns `null` if not, avoiding exceptions for expected scenarios
- This keeps the notification object always valid—if you have an instance, the underlying resource exists

```php
abstract class AbstractPushNotification {
    protected int $resource_id;
    protected array $tokens = [];

    public function __construct(int $resource_id) {
        $this->resource_id = $resource_id;
    }

    abstract public function get_type(): string;
    abstract public function get_title(): string;
    abstract public function get_message(): string;
    abstract public function get_icon(): string;
    abstract protected function get_meta(): array;

    /**
     * Set the push tokens for this notification.
     *
     * @param PushToken[] $tokens Array of PushToken instances.
     */
    public function set_tokens(array $tokens): void {
        $this->tokens = $tokens;
    }

    /**
     * Generate a unique idempotency key for this notification.
     * WPCOM uses this to deduplicate if the same notification is sent twice
     * (e.g., due to DB replication lag in multi-server environments).
     */
    public function get_idempotency_key(): string {
        return $this->get_type() . '_' . $this->resource_id;
    }

    /**
     * Factory method to create the appropriate notification instance.
     *
     * @param string $type        Notification type ('store_order', 'store_review').
     * @param int    $resource_id Order ID or comment ID.
     * @return static|null The notification instance, or null if resource doesn't exist.
     */
    public static function create_from_type(string $type, int $resource_id): ?self {
        return match ($type) {
            'store_order'  => NewOrderPushNotification::create($resource_id),
            'store_review' => NewReviewPushNotification::create($resource_id),
            default        => null,
        };
    }

    /**
     * Build the API payload.
     */
    public function to_array(): array {
        return array(
            'idempotency_key' => $this->get_idempotency_key(),
            'type'            => $this->get_type(),
            'title'           => $this->get_title(),
            'message'         => $this->get_message(),
            'icon'            => $this->get_icon(),
            'timestamp'       => gmdate('c'),
            'resource_id'     => $this->resource_id,
            'meta'            => $this->get_meta(),
            'tokens'          => array_map(
                fn(PushToken $token) => array(
                    'user_id'     => $token->get_user_id(),
                    'token'       => $token->get_token(),
                    'platform'    => $token->get_platform(),
                    'environment' => $token->get_environment(),
                ),
                $this->tokens
            ),
        );
    }
}
```

```php
class NewOrderPushNotification extends AbstractPushNotification {
    private WC_Order $order;

    private function __construct(int $order_id, WC_Order $order) {
        parent::__construct($order_id);
        $this->order = $order;
    }

    /**
     * Create a notification instance, or return null if order doesn't exist.
     */
    public static function create(int $order_id): ?self {
        // wc_get_order() uses caching and handles HPOS/posts abstraction.
        $order = wc_get_order($order_id);
        if (!$order) {
            return null;
        }
        return new self($order_id, $order);
    }

    public function get_type(): string {
        return 'store_order';
    }

    public function get_title(): string {
        return __('New Order', 'woocommerce');
    }

    public function get_message(): string {
        // Use WC_Order methods - these read from the already-loaded object,
        // no additional DB queries. get_order_number() returns the display
        // number (may differ from ID if customized).
        return sprintf(
            /* translators: 1: order number, 2: customer name */
            __('Order #%1$s received from %2$s', 'woocommerce'),
            $this->order->get_order_number(),
            $this->order->get_formatted_billing_full_name()
        );
    }

    public function get_icon(): string {
        return 'https://example.com/icons/new-order.png'; // TODO: Real URL
    }

    protected function get_meta(): array {
        // get_total() reads from already-loaded order data, no additional queries.
        // Note: currency removed per WPCOM requirements.
        return array(
            'order_total' => $this->order->get_total(),
        );
    }
}
```

```php
class NewReviewPushNotification extends AbstractPushNotification {
    private WP_Comment $comment;
    private WC_Product $product;

    private function __construct(int $comment_id, WP_Comment $comment, WC_Product $product) {
        parent::__construct($comment_id);
        $this->comment = $comment;
        $this->product = $product;
    }

    /**
     * Create a notification instance, or return null if comment/product doesn't exist.
     */
    public static function create(int $comment_id): ?self {
        // get_comment() uses WordPress comment cache.
        $comment = get_comment($comment_id);
        if (!$comment) {
            return null;
        }

        // wc_get_product() uses caching and handles product type polymorphism.
        $product = wc_get_product($comment->comment_post_ID);
        if (!$product) {
            return null;
        }

        return new self($comment_id, $comment, $product);
    }

    public function get_type(): string {
        return 'store_review';
    }

    public function get_title(): string {
        return __('New Review', 'woocommerce');
    }

    public function get_message(): string {
        // comment_author is a direct property (no DB query).
        // get_name() reads from already-loaded product data.
        return sprintf(
            /* translators: 1: reviewer name, 2: product name */
            __('%1$s left a review on %2$s', 'woocommerce'),
            $this->comment->comment_author,
            $this->product->get_name()
        );
    }

    public function get_icon(): string {
        return 'https://example.com/icons/new-review.png'; // TODO: Real URL
    }

    protected function get_meta(): array {
        // Note: rating removed per WPCOM requirements.
        return array(
            'product_id' => $this->product->get_id(),
        );
    }
}
```

### 2. PushNotificationSender Service

Makes HTTP requests to WordPress.com API using Jetpack site token for authorization:

```php
class PushNotificationSender {
    const API_ENDPOINT = 'https://public-api.wordpress.com/wpcom/v2/push-notifications';

    /**
     * Send a notification to WordPress.com.
     *
     * @param AbstractPushNotification $notification The notification to send.
     * @return true|WP_Error True on success, WP_Error on failure.
     */
    public function send(AbstractPushNotification $notification): bool|WP_Error {
        $token = $this->get_jetpack_site_token();

        if (is_wp_error($token)) {
            return $token;
        }

        $response = wp_remote_post(
            self::API_ENDPOINT,
            array(
                'timeout' => 30,
                'headers' => array(
                    'Authorization' => 'X-Jetpack ' . $token,
                    'Content-Type'  => 'application/json',
                ),
                'body' => wp_json_encode($notification->to_array()),
            )
        );

        if (is_wp_error($response)) {
            return $response;
        }

        $status_code = wp_remote_retrieve_response_code($response);

        // WPCOM returns 201 when the notification has been queued for delivery.
        // (No guarantee of delivery, and no way to check outcome.)
        if ($status_code !== 201) {
            $body = wp_remote_retrieve_body($response);
            return new WP_Error(
                'push_notification_failed',
                sprintf('WordPress.com API returned %d: %s', $status_code, $body),
                array('status' => $status_code)
            );
        }

        return true;
    }

    /**
     * Get the Jetpack site token for authorization.
     *
     * @return string|WP_Error The token or error.
     */
    private function get_jetpack_site_token(): string|WP_Error {
        if (!class_exists('Automattic\Jetpack\Connection\Tokens')) {
            return new WP_Error(
                'jetpack_not_available',
                'Jetpack connection is not available'
            );
        }

        $tokens = new \Automattic\Jetpack\Connection\Tokens();
        $token = $tokens->get_access_token();

        if (!$token || empty($token->secret)) {
            return new WP_Error(
                'jetpack_not_connected',
                'Site is not connected to Jetpack'
            );
        }

        return $token->secret;
    }
}
```

### 3. PushNotificationRestController (REST Endpoint for Sending Notifications)

**Important considerations from team review:**
1. Loopback requests can fail → use ActionScheduler as fallback
2. Don't pass cookies (Jetpack had issues with this)
3. Enforce time limit on async endpoint
4. Use HMAC-signed tokens for stateless auth (avoids DB replication lag)
5. Primary duplicate prevention is in trigger (atomic meta write)
6. Include idempotency key in notification for WPCOM-side deduplication
7. Return 201 Created (matches WPCOM semantics: queued for delivery, no delivery guarantee)

```php
class PushNotificationRestController extends RestApiControllerBase {
    const TIME_LIMIT = 30; // seconds

    protected string $route_namespace = 'wc-push-notifications';
    protected string $rest_base = 'async';

    private PushNotificationLoopbackSpawner $spawner;
    private PushNotificationSender $push_notification_sender;
    private PushNotificationJobManager $job_manager;
    private PushTokensDataStore $push_tokens_data_store;

    public function register_routes(): void {
        register_rest_route(
            $this->get_rest_api_namespace(),
            $this->rest_base . '/send',
            array(
                'methods'             => WP_REST_Server::CREATABLE,
                'callback'            => fn($request) => $this->run($request, 'create'),
                'permission_callback' => array($this, 'authorize'),
            )
        );
    }

    /**
     * Authenticate the async request using HMAC signature.
     *
     * This is the authentication mechanism for the loopback endpoint. External users
     * cannot access this endpoint because they cannot forge valid HMAC tokens without
     * knowing the server's secret salts (wp_salt('auth')).
     *
     * Similar pattern to StoreApi/Utilities/JsonWebToken.php which uses HMAC-SHA256
     * for stateless authentication of cart tokens.
     *
     * @see \Automattic\WooCommerce\StoreApi\Utilities\JsonWebToken
     */
    public function authorize(WP_REST_Request $request): bool|WP_Error {
        $token = $request->get_header('X-WC-Push-Token');

        if (!$token) {
            return new WP_Error('missing_token', 'Missing authentication token', array('status' => 401));
        }

        $body = $request->get_json_params();
        $expected_data = array(
            'type'        => $body['type'] ?? '',
            'resource_id' => (int) ($body['resource_id'] ?? 0),
        );

        if (!$this->spawner->verify_hmac_token($token, $expected_data)) {
            return new WP_Error('invalid_token', 'Invalid or expired token', array('status' => 401));
        }

        return true;
    }

    public function create(WP_REST_Request $request): WP_REST_Response {
        // Enforce time limit
        set_time_limit(self::TIME_LIMIT);

        // Release PHP session lock to allow concurrent requests.
        //
        // PHP sessions use file-based locking by default. While a session is open,
        // other requests from the same user (same session ID) are blocked waiting
        // for the lock. Since this async endpoint may take time to complete (sending
        // to WPCOM API, potential retries), keeping the session open would block
        // the user's other requests.
        //
        // session_write_close() writes session data and releases the lock without
        // ending the script. After this call:
        // - $_SESSION is still readable but changes won't be saved
        // - Other requests from this user can proceed
        //
        // Side effects if NOT done: User's browser may appear frozen if they have
        // multiple tabs/requests in flight, as all would queue behind this request.
        if (session_id()) {
            session_write_close();
        }

        $body = $request->get_json_params();
        $type = sanitize_text_field($body['type'] ?? '');
        $resource_id = absint($body['resource_id'] ?? 0);

        // 1. Build the notification object (validates resource exists)
        $notification = AbstractPushNotification::create_from_type($type, $resource_id);

        if (!$notification) {
            // Resource doesn't exist - nothing to do, return 201 anyway
            // (fire-and-forget, caller doesn't check response)
            return new WP_REST_Response(null, 201);
        }

        // 2. Get tokens for admin/shop_manager users
        $tokens = $this->get_tokens_for_notification();
        $notification->set_tokens($tokens);

        // 3. Send to WordPress.com
        $result = $this->push_notification_sender->send($notification);

        // 4. Schedule retry if failed
        if (is_wp_error($result)) {
            // Log the error for debugging (note 10)
            wc_get_logger()->error(
                sprintf(
                    'Push notification failed for %s %d: %s',
                    $type,
                    $resource_id,
                    $result->get_error_message()
                ),
                array('source' => 'wc-push-notifications')
            );

            $this->job_manager->schedule_retry(
                array('type' => $type, 'resource_id' => $resource_id),
                1
            );
        }

        // Return 201 to match WPCOM semantics: queued for delivery, no delivery guarantee.
        return new WP_REST_Response(null, 201);
    }

    private function get_tokens_for_notification(): array {
        // Query tokens for users with administrator/shop_manager roles
        return $this->push_tokens_data_store->get_by_owner_roles(
            PushNotifications::ROLES_WITH_PUSH_NOTIFICATIONS_ENABLED
        );
    }
}
```

### 3b. PushNotificationLoopbackSpawner (Spawns Async Requests)

Uses HMAC-signed tokens for authentication, following the same pattern as
`StoreApi/Utilities/JsonWebToken.php`. The token:
- Is signed with `wp_salt('auth')` which only the server knows
- Contains an expiration timestamp to prevent replay attacks
- Contains the request data (type, resource_id) to prevent token reuse for different resources
- Contains `site_url` to prevent cross-site use in multisite/cloning scenarios

This provides stateless authentication without cookies or database lookups,
avoiding both Jetpack's cookie issues and database replication lag.

```php
class PushNotificationLoopbackSpawner {
    const TOKEN_EXPIRY = 300; // 5 minutes

    /**
     * Spawn an async request to the internal endpoint.
     * If the loopback fails, schedule via ActionScheduler as fallback.
     *
     * @param string $endpoint   The endpoint to call (e.g., 'send').
     * @param array  $data       The data to send (includes type, resource_id).
     */
    public function spawn(string $endpoint, array $data): void {
        $token = $this->generate_hmac_token($data);

        $url = rest_url('wc-push-notifications/async/' . $endpoint);

        $response = wp_remote_post($url, array(
            'timeout'   => 0.01,
            'blocking'  => false,
            'body'      => wp_json_encode($data),
            'headers'   => array(
                'Content-Type'     => 'application/json',
                'X-WC-Push-Token'  => $token,
            ),
            // SSL verification for loopback requests:
            //
            // Local/loopback requests often fail SSL verification because:
            // 1. Development environments use self-signed certificates
            // 2. The server's SSL certificate may not include localhost/127.0.0.1 as a valid SAN
            // 3. Some hosting environments have certificate chain issues for internal requests
            //
            // This is safe because:
            // - The request stays within the same server (loopback to self)
            // - The HMAC token provides authentication (an attacker can't forge valid tokens)
            // - No sensitive data is transmitted that isn't already on this server
            //
            // The filter allows hosts to enable verification if their environment supports it.
            'sslverify' => apply_filters('https_local_ssl_verify', false),
            // NO cookies - Jetpack had issues with this
        ));

        // If loopback request failed immediately, schedule via ActionScheduler
        if (is_wp_error($response)) {
            $this->schedule_fallback($endpoint, $data);
        }
    }

    /**
     * Generate an HMAC-signed token for stateless authentication.
     *
     * Token format: base64(payload).signature
     * Payload includes expiry, type, resource_id, and site_url for verification.
     * No database storage needed - receiving server verifies signature.
     */
    private function generate_hmac_token(array $data): string {
        $payload = wp_json_encode(array(
            'type'        => $data['type'] ?? 'unknown',
            'resource_id' => $data['resource_id'] ?? 0,
            'expires'     => time() + self::TOKEN_EXPIRY,
            'site_url'    => site_url(), // Bind token to this site (multisite/cloning protection)
        ));

        $signature = hash_hmac('sha256', $payload, wp_salt('auth'));

        return base64_encode($payload) . '.' . $signature;
    }

    /**
     * Verify an HMAC-signed token.
     *
     * @param string $token The token to verify.
     * @param array  $expected_data The expected type and resource_id.
     * @return bool True if valid, false otherwise.
     */
    public function verify_hmac_token(string $token, array $expected_data): bool {
        $parts = explode('.', $token);
        if (count($parts) !== 2) {
            return false;
        }

        list($payload_b64, $signature) = $parts;
        $payload = base64_decode($payload_b64);

        if (!$payload) {
            return false;
        }

        // Verify signature (timing-safe comparison)
        $expected_signature = hash_hmac('sha256', $payload, wp_salt('auth'));
        if (!hash_equals($expected_signature, $signature)) {
            return false;
        }

        // Verify payload contents
        $data = json_decode($payload, true);
        if (!$data) {
            return false;
        }

        // Check expiry
        if (($data['expires'] ?? 0) < time()) {
            return false;
        }

        // Verify site_url matches (multisite/cloning protection)
        if (($data['site_url'] ?? '') !== site_url()) {
            return false;
        }

        // Verify type and resource_id match the request body
        if (($data['type'] ?? '') !== ($expected_data['type'] ?? '')) {
            return false;
        }
        if (($data['resource_id'] ?? 0) !== ($expected_data['resource_id'] ?? 0)) {
            return false;
        }

        return true;
    }

    /**
     * Fallback to ActionScheduler if loopback fails.
     *
     * Instead of a separate fallback processor, we schedule a retry with attempt=0
     * which means "immediate". The PushNotificationRetryProcessor handles all send
     * attempts, whether the failure was at the loopback level or the WPCOM API level.
     */
    private function schedule_fallback(string $endpoint, array $data): void {
        $this->job_manager->schedule_retry(
            array(
                'type'        => $data['type'] ?? '',
                'resource_id' => $data['resource_id'] ?? 0,
            ),
            0 // attempt=0 means immediate execution
        );
    }

    private PushNotificationJobManager $job_manager;
}
```

### 4. PushNotificationJobManager

Manages ActionScheduler jobs for retries:

```php
class PushNotificationJobManager {
    const RETRY_HOOK = 'wc_push_notification_retry';
    const GROUP = 'wc-push-notifications';

    // Retry delays in seconds: 10s, 60s, 5m, 15m, 60m
    const RETRY_DELAYS = array(10, 60, 300, 900, 3600);
    const MAX_RETRIES = 5;

    /**
     * Schedule a retry for a failed notification.
     *
     * @param array $args    Notification data (type, resource_id).
     * @param int   $attempt Attempt number. 0 = immediate (for loopback failures),
     *                       1-5 = delayed retries (for WPCOM API failures).
     */
    public function schedule_retry(array $args, int $attempt): void {
        if ($attempt > self::MAX_RETRIES) {
            $this->log_failure($args);
            return;
        }

        // attempt=0 means immediate (for loopback failures)
        // attempt=1+ uses the delay schedule
        if ($attempt === 0) {
            $delay = 0;
            $args['attempt'] = 1; // Track as attempt 1 going forward
        } else {
            $delay = self::RETRY_DELAYS[$attempt - 1] ?? self::RETRY_DELAYS[self::MAX_RETRIES - 1];
            $args['attempt'] = $attempt;
        }

        WC()->queue()->schedule_single(
            time() + $delay,
            self::RETRY_HOOK,
            array($args),
            self::GROUP
        );
    }
}
```

### 5. PushNotificationRetryProcessor

Processes retry jobs from ActionScheduler.

**Why no deduplication check here?**
The meta flag is set at trigger time to *claim* the notification, not to indicate successful delivery.
If a retry is scheduled, the previous send attempt failed, so we should attempt again.
WPCOM's `idempotency_key` handles the edge case where a notification is somehow delivered twice.

```php
class PushNotificationRetryProcessor {
    private PushNotificationSender $notification_sender;
    private PushNotificationJobManager $job_manager;
    private PushTokensDataStore $push_tokens_data_store;

    public function __construct() {
        add_action(PushNotificationJobManager::RETRY_HOOK, array($this, 'process'));
    }

    public function process(array $args): void {
        $type = $args['type'] ?? '';
        $resource_id = $args['resource_id'] ?? 0;
        $attempt = $args['attempt'] ?? 1;

        // Rebuild notification (validates resource still exists)
        $notification = AbstractPushNotification::create_from_type($type, $resource_id);
        if (!$notification) {
            // Resource no longer exists - nothing to do
            return;
        }

        // Get fresh tokens (user may have registered new devices)
        $tokens = $this->push_tokens_data_store->get_by_owner_roles(
            PushNotifications::ROLES_WITH_PUSH_NOTIFICATIONS_ENABLED
        );
        $notification->set_tokens($tokens);

        $result = $this->notification_sender->send($notification);

        if (is_wp_error($result)) {
            wc_get_logger()->error(
                sprintf(
                    'Push notification retry %d failed for %s %d: %s',
                    $attempt,
                    $type,
                    $resource_id,
                    $result->get_error_message()
                ),
                array('source' => 'wc-push-notifications')
            );

            $this->job_manager->schedule_retry($args, $attempt + 1);
        }
    }
}
```

### 6. NewOrderPushNotificationTrigger

Triggers notification on new orders. Uses two hooks because:
- `woocommerce_new_order` - Catches orders created directly with a qualifying status
- `woocommerce_order_status_changed` - Catches orders that transition to a qualifying status later

Note: `woocommerce_order_status_changed` only fires when there's a "from" status (see `class-wc-order.php:470`), so orders created directly with 'processing' status won't trigger it.

**HPOS Sync Mode Consideration:**
When HPOS sync mode is enabled, Jetpack Sync imports orders from BOTH wp_posts AND
wc_orders tables separately. Each sync checks its own table for the meta flag. To prevent
duplicates, we must store the meta in BOTH tables when sync mode is enabled.

```php
class NewOrderPushNotificationTrigger extends AbstractPushNotificationTrigger {
    /**
     * WooCommerce-specific meta key - this is OUR source of truth.
     * Used to track whether WC has sent a push notification for this order.
     */
    const META_KEY_NOTIFIED = '_wc_push_notification_sent';

    /**
     * Legacy Jetpack Sync compatible key.
     * We write to this for backwards compatibility with Jetpack Sync's deduplication,
     * but we don't read from it (our source of truth is META_KEY_NOTIFIED).
     * This can be removed once Jetpack Sync is no longer using it for push notifications.
     */
    const META_KEY_JETPACK_COMPAT = '_wpcom_new_order_note_created';

    const NOTIFY_STATUSES = array(
        'processing',
        'on-hold',
        'completed',
        'pre-order',
        'pre-ordered',
        'partial-payment',
    );

    public function register(): void {
        // For orders created directly with a qualifying status (API, admin, etc.)
        add_action('woocommerce_new_order', array($this, 'on_new_order'), 10, 2);

        // For orders that transition to a qualifying status
        add_action('woocommerce_order_status_changed', array($this, 'on_order_status_changed'), 10, 3);
    }

    public function on_new_order(int $order_id, WC_Order $order): void {
        if (!in_array($order->get_status(), self::NOTIFY_STATUSES, true)) {
            return;
        }

        $this->maybe_send_notification($order);
    }

    public function on_order_status_changed(int $order_id, string $old_status, string $new_status): void {
        if (!in_array($new_status, self::NOTIFY_STATUSES, true)) {
            return;
        }

        $order = wc_get_order($order_id);
        if (!$order) {
            return;
        }
        $this->maybe_send_notification($order);
    }

    /**
     * Check and mark notification for this order.
     *
     * Uses the same pattern as stock reduction in wc-stock-functions.php:
     * check flag, then set flag. This is NOT truly atomic (small race window
     * exists between check and set), but the consequence is minor (extra HTTP
     * request) and WPCOM deduplicates via idempotency_key.
     *
     * @see wc_maybe_reduce_stock_levels() for the pattern we're following.
     */
    private function maybe_send_notification(WC_Order $order): void {
        // Check if already notified (similar to get_stock_reduced check)
        if ($this->has_been_notified($order)) {
            return;
        }

        // Mark as notified (similar to set_stock_reduced)
        $this->mark_as_notified($order);

        // Queue for sending on shutdown
        $this->queue_notification(
            'store_order',
            $order->get_id()
        );
    }

    /**
     * Check if notification has already been claimed for this order.
     * Checks both WC and Jetpack compat keys in both HPOS and postmeta.
     */
    private function has_been_notified(WC_Order $order): bool {
        // Check WC source of truth via order object (reads from active datastore)
        if ($order->get_meta(self::META_KEY_NOTIFIED)) {
            return true;
        }

        // Check Jetpack compat key via order object
        if ($order->get_meta(self::META_KEY_JETPACK_COMPAT)) {
            return true;
        }

        // Also check postmeta directly (covers Jetpack Sync having processed this)
        if (get_post_meta($order->get_id(), self::META_KEY_NOTIFIED, true)) {
            return true;
        }
        if (get_post_meta($order->get_id(), self::META_KEY_JETPACK_COMPAT, true)) {
            return true;
        }

        return false;
    }

    /**
     * Mark order as having been claimed for notification.
     *
     * Writes both keys via order object (goes to active datastore).
     * When sync mode is enabled, also writes directly to postmeta because
     * Jetpack Sync imports from both HPOS and posts tables separately.
     */
    private function mark_as_notified(WC_Order $order): void {
        $timestamp = time();

        // Store both keys via order object (goes to active datastore - HPOS or posts)
        $order->update_meta_data(self::META_KEY_NOTIFIED, $timestamp);
        $order->update_meta_data(self::META_KEY_JETPACK_COMPAT, $timestamp);
        $order->save();

        // When sync mode is enabled, also write directly to postmeta
        // (Jetpack Sync imports from posts table separately)
        if ($this->is_sync_mode_enabled()) {
            update_post_meta($order->get_id(), self::META_KEY_NOTIFIED, $timestamp);
            update_post_meta($order->get_id(), self::META_KEY_JETPACK_COMPAT, $timestamp);
        }
    }

    /**
     * Check if HPOS sync mode is enabled (data in both tables).
     */
    private function is_sync_mode_enabled(): bool {
        $data_sync = wc_get_container()->get(DataSynchronizer::class);
        return $data_sync->data_sync_is_enabled();
    }
}
```

### 7. NewReviewPushNotificationTrigger

Triggers notification on new reviews:

```php
class NewReviewPushNotificationTrigger extends AbstractPushNotificationTrigger {
    const META_KEY_NOTIFIED = '_wc_push_notification_sent';

    public function register(): void {
        add_action('comment_post', array($this, 'on_comment_post'), 10, 3);
    }

    public function on_comment_post(int $comment_id, int|string $approved, array $commentdata): void {
        // Check if this is a product review
        if (($commentdata['comment_type'] ?? '') !== 'review') {
            return;
        }

        // Atomically try to claim this notification.
        // add_comment_meta with $unique=true returns false if key already exists.
        $claimed = add_comment_meta(
            $comment_id,
            self::META_KEY_NOTIFIED,
            time(),
            true // unique - fails if already exists
        );

        if (!$claimed) {
            return; // Another process already claimed this
        }

        // Queue for sending on shutdown
        $this->queue_notification(
            'store_review',
            $comment_id
        );
    }
}
```

### 8. AbstractPushNotificationTrigger Base Class

Common functionality for triggers.

**Important: Uses static properties to ensure all trigger instances share the same pending queue.**
If NewOrderPushNotificationTrigger and NewReviewPushNotificationTrigger both fire in the same request,
they must add to the same queue so all notifications are sent together at shutdown.

```php
abstract class AbstractPushNotificationTrigger {
    /**
     * Shared pending notifications queue.
     * Static so all trigger instances share the same queue within a request.
     *
     * @var array<array{type: string, resource_id: int}>
     */
    private static array $pending = array();

    /**
     * Whether shutdown hook is registered.
     * Static to prevent multiple registrations across instances.
     */
    private static bool $shutdown_registered = false;

    protected PushNotificationLoopbackSpawner $spawner;

    protected function queue_notification(string $type, int $resource_id): void {
        self::$pending[] = array(
            'type'        => $type,
            'resource_id' => $resource_id,
        );

        if (!self::$shutdown_registered) {
            // Use the abstract class name explicitly, not self::class.
            // With self::class (late static binding), this would resolve to the
            // child class name (e.g., NewOrderPushNotificationTrigger), which works
            // but is confusing since send_pending() is defined here.
            add_action('shutdown', array(__CLASS__, 'send_pending'));
            self::$shutdown_registered = true;
        }
    }

    /**
     * Send all pending notifications at shutdown.
     * Static method because it's called via add_action with class reference.
     */
    public static function send_pending(): void {
        $spawner = wc_get_container()->get(PushNotificationLoopbackSpawner::class);

        foreach (self::$pending as $notification) {
            $spawner->spawn('send', $notification);
        }

        // Clear for safety (static state persists in long-running processes)
        self::$pending = array();
        self::$shutdown_registered = false;
    }

    abstract public function register(): void;
}
```

## WordPress.com API Payload

```json
{
    "idempotency_key": "store_order_123",
    "type": "store_order",
    "title": "New Order",
    "message": "Order #123 received from John Doe",
    "icon": "https://example.com/icon.png",
    "timestamp": "2024-01-15T12:00:00Z",
    "resource_id": 123,
    "meta": {
        "order_total": "99.99"
    },
    "tokens": [
        {
            "user_id": 1,
            "token": "abc123...",
            "platform": "ios",
            "environment": "production"
        }
    ]
}
```

The `idempotency_key` allows WPCOM to deduplicate notifications. If the same
key is received within a time window, WPCOM can ignore the duplicate.

## Retry Strategy

| Attempt | Delay | Total Time Elapsed |
|---------|-------|-------------------|
| 1       | 10s   | 10s               |
| 2       | 60s   | 1m 10s            |
| 3       | 5m    | 6m 10s            |
| 4       | 15m   | 21m 10s           |
| 5       | 60m   | 1h 21m 10s        |

After 5 failed attempts, the notification is logged and abandoned.

## Tests Required

### Unit Tests

1. **AbstractPushNotificationTest** (and concrete implementations)
   - Validation of required fields
   - Abstract method enforcement
   - to_array() output format
   - create_from_type() returns correct subclass for each type
   - create_from_type() returns null for unknown type
   - create_from_type() returns null if resource doesn't exist

2. **PushNotificationSenderTest**
   - Successful API response handling
   - Error response handling
   - WP_Error handling
   - Authorization header format

3. **PushNotificationRestControllerTest**
   - HMAC token verification (valid, expired, tampered, mismatched data)
   - Payload building for orders
   - Payload building for reviews
   - Token retrieval
   - Retry scheduling on failure

4. **PushNotificationLoopbackSpawnerTest**
   - HMAC token generation includes site_url
   - HMAC token verification (valid, expired, tampered, wrong site_url)
   - Fallback scheduling when loopback fails (calls schedule_retry with attempt=0)

5. **PushNotificationJobManagerTest**
   - Retry scheduling with correct delays
   - Attempt=0 schedules immediately (no delay)
   - Attempt=0 is tracked as attempt=1 going forward
   - Max retry limit enforcement
   - ActionScheduler integration

6. **PushNotificationRetryProcessorTest**
   - Rebuilds notification for valid resources
   - Skips if resource no longer exists
   - Gets fresh tokens on each retry
   - Schedules next retry on failure
   - Logs errors via wc_get_logger()

7. **NewOrderPushNotificationTriggerTest**
   - Duplicate prevention via meta check
   - Dual-write (WC key + Jetpack compat key)
   - HPOS sync mode handling
   - Status change detection
   - Only triggers for specified statuses
   - Shutdown hook registration
   - Null order handling in on_order_status_changed

8. **NewReviewPushNotificationTriggerTest**
   - Comment type filtering (only reviews)
   - Atomic meta write for duplicate prevention
   - Shutdown hook registration

9. **PushTokenTest** (addition to existing tests)
   - get_environment() returns 'sandbox' for ':dev' origins
   - get_environment() returns 'production' for non-':dev' origins

## Resolved Questions

1. **Icon URL**: Static strings per notification type, defined in each notification class.
2. **Environment**: Determined by token origin via `PushToken::get_environment()` - if origin ends in `:dev` → `'sandbox'`, otherwise `'production'`. (WPCOM terminology)
3. **Authorization**: Use Jetpack connection **site token** (not user token).
4. **Notification Content**: A class per notification type (like WPCOM), each encapsulating its own title/message/icon.
5. **Token representation**: Tokens are `PushToken` instances (not arrays). The `get_environment()` method lives on `PushToken`.
6. **Meta fields**: Order notifications include `order_total` only (no currency). Review notifications include `product_id` only (no rating).
7. **WPCOM response**: WPCOM returns 201 when queued. Our endpoint also returns 201 to match semantics.
8. **Order meta keys**: Two keys stored - `_wc_push_notification_sent` (WC source of truth) and `_wpcom_new_order_note_created` (Jetpack Sync compat).
9. **Loopback authentication**: HMAC-signed tokens authenticate requests to the internal endpoint. External users cannot forge valid tokens without access to `wp_salt('auth')`. Tokens include `site_url` to prevent cross-site use in multisite/cloning scenarios. This follows the same pattern as `StoreApi/Utilities/JsonWebToken.php`.
