# X API Core Concepts

Reference for the underlying X API v2 concepts that both XDKs build on.

## Response Structure

Every X API v2 response follows this pattern:

```json
{
  "data": { ... },        // Primary object(s) - single object or array
  "includes": {           // Expanded related objects (when expansions requested)
    "users": [...],
    "tweets": [...],
    "media": [...],
    "polls": [...],
    "places": [...]
  },
  "meta": {               // Pagination and result metadata
    "result_count": 10,
    "next_token": "abc123",
    "previous_token": "xyz789"
  },
  "errors": [...]         // Partial errors (some data may still be returned)
}
```

## Fields Reference

The API returns minimal data by default. Request additional fields per object type:

| Parameter | Default Fields | Common Additional Fields |
|---|---|---|
| `tweet.fields` | `id`, `text`, `edit_history_tweet_ids` | `created_at`, `author_id`, `public_metrics`, `conversation_id`, `lang`, `possibly_sensitive`, `referenced_tweets`, `attachments` |
| `user.fields` | `id`, `name`, `username` | `created_at`, `description`, `location`, `public_metrics`, `profile_image_url`, `verified` |
| `media.fields` | `media_key`, `type` | `url`, `preview_image_url`, `alt_text`, `public_metrics`, `duration_ms`, `width`, `height` |
| `poll.fields` | `id` | `options`, `end_datetime`, `voting_status`, `duration_minutes` |
| `place.fields` | `id`, `full_name` | `country`, `country_code`, `geo`, `name`, `place_type` |

**Important**: You cannot request subfields. `public_metrics` returns all metrics (likes, reposts, replies, quotes).

## Expansions Reference

Expansions include related objects in the `includes` section. Link them via IDs.

### Post Expansions

| Expansion | Returns | Use Case |
|---|---|---|
| `author_id` | User | Post author details |
| `referenced_tweets.id` | Post(s) | Quoted/replied-to posts |
| `referenced_tweets.id.author_id` | User(s) | Authors of referenced posts |
| `in_reply_to_user_id` | User | User being replied to |
| `attachments.media_keys` | Media | Images, videos, GIFs |
| `attachments.poll_ids` | Poll | Poll options and votes |
| `geo.place_id` | Place | Location details |
| `entities.mentions.username` | User(s) | Mentioned users |

### User Expansions

| Expansion | Returns | Use Case |
|---|---|---|
| `pinned_tweet_id` | Post | User's pinned post |

### DM Expansions

| Expansion | Returns | Use Case |
|---|---|---|
| `sender_id` | User | Message sender |
| `participant_ids` | User(s) | Conversation participants |
| `attachments.media_keys` | Media | Attached media |

## Common Field + Expansion Combinations

**Post analytics:**
```
tweet.fields=created_at,public_metrics,possibly_sensitive
```

**Full post context:**
```
tweet.fields=created_at,author_id,conversation_id,in_reply_to_user_id,referenced_tweets
expansions=author_id,referenced_tweets.id,attachments.media_keys
user.fields=username,name,profile_image_url
media.fields=url,preview_image_url,type,alt_text
```

**User profiles:**
```
user.fields=created_at,description,location,public_metrics,verified,profile_image_url
```

## Pagination

Token-based. Results returned in reverse chronological order.

| Concept | Description |
|---|---|
| `max_results` | Results per page (endpoint-specific max, typically 100) |
| `next_token` | In response `meta` - use for next page |
| `previous_token` | In response `meta` - use to go back |
| `pagination_token` | Request parameter - set to `next_token` value |

Pagination stops when `next_token` is absent from the response.

## Authentication Methods

| Method | Use Case | SDK Config |
|---|---|---|
| **Bearer Token** | App-only, read-only public data | `bearerToken` (TS) / `bearer_token` (PY) |
| **OAuth 2.0 PKCE** | User-context with scopes (recommended) | `accessToken` (TS) / `bearer_token` with access token (PY) |
| **OAuth 1.0a** | Legacy user-context | `oauth1` (TS) / `auth` (PY) |

### Common OAuth 2.0 Scopes

| Scope | Access |
|---|---|
| `tweet.read` | Read posts |
| `tweet.write` | Create/delete posts |
| `users.read` | Read user profiles |
| `follows.read` | Read follow relationships |
| `follows.write` | Follow/unfollow |
| `like.read` | Read likes |
| `like.write` | Like/unlike |
| `dm.read` | Read DMs |
| `dm.write` | Send DMs |
| `offline.access` | Get refresh token |
| `space.read` | Read Spaces |
| `list.read` / `list.write` | Read/manage lists |
| `bookmark.read` / `bookmark.write` | Read/manage bookmarks |

## Rate Limits

Limits are per 15-minute window unless noted. Key limits:

| Endpoint | Per App | Per User |
|---|---|---|
| `GET /2/tweets/:id` | 450 | 900 |
| `GET /2/tweets` (batch) | 3,500 | 5,000 |
| `GET /2/tweets/search/recent` | 450 | 300 |
| `GET /2/tweets/search/all` | 300 (1/sec) | 1/sec |
| `POST /2/tweets` | 10,000/24hrs | 100 |
| `GET /2/users/:id` | 300 | 900 |
| `GET /2/users/:id/followers` | 300 | 300 |
| `GET /2/tweets/search/stream` | 50 | - |
| `POST /2/media/upload` | 50,000/24hrs | 500 |

### Rate Limit Headers

```
x-rate-limit-limit: 900         # Max requests in window
x-rate-limit-remaining: 847     # Remaining requests
x-rate-limit-reset: 1705420800  # Unix timestamp when window resets
```

### Recovery Strategy

1. Check `x-rate-limit-reset` header
2. Wait until reset time
3. Use exponential backoff on 429 errors

## Search Query Operators

Used with `client.posts.searchRecent()` / `client.posts.search_recent()`:

| Operator | Example | Description |
|---|---|---|
| keyword | `python` | Posts containing word |
| `"exact phrase"` | `"hello world"` | Exact phrase match |
| `from:` | `from:xdevelopers` | Posts by user |
| `to:` | `to:xdevelopers` | Replies to user |
| `@` | `@xdevelopers` | Mentions user |
| `#` | `#typescript` | Contains hashtag |
| `has:images` | `has:images` | Posts with images |
| `has:videos` | `has:videos` | Posts with video |
| `has:media` | `has:media` | Posts with any media |
| `has:links` | `has:links` | Posts with links |
| `is:retweet` | `is:retweet` | Only retweets |
| `-is:retweet` | `-is:retweet` | Exclude retweets |
| `is:reply` | `is:reply` | Only replies |
| `-is:reply` | `-is:reply` | Exclude replies |
| `lang:` | `lang:en` | Posts in language |
| `url:` | `url:github.com` | Posts linking to URL |
| `conversation_id:` | `conversation_id:123` | Posts in thread |
| `context:` | `context:123.456` | Posts matching context annotation |
| `entity:` | `entity:"Bitcoin"` | Posts with entity annotation |
| `place:` | `place:"New York"` | Posts from location |

Operators can be combined: `from:xdevelopers -is:retweet has:media lang:en`

Max query length: 512 chars (recent search), 1024 chars (full-archive).

## Streaming

### Stream Types

| Type | Endpoint | Rules | Volume | Connections |
|---|---|---|---|---|
| **Filtered Stream** | `GET /2/tweets/search/stream` | Required | Matching posts only | 1 (pay-per-use), 2+ (Enterprise) |
| **Sampled Stream (1%)** | `GET /2/tweets/sample/stream` | None | ~1% random | 1 |
| **Sampled Stream (10%)** | `GET /2/tweets/sample10/stream` | None | ~10% random | Enterprise |
| **Firehose** | `GET /2/tweets/firehose/stream` | None | All posts | Enterprise |
| **Language Firehose** | `/2/tweets/firehose/stream/lang/{lang}` | None | All posts in language | Enterprise |

### Filtered Stream: Rules Are Mandatory

**The filtered stream delivers NO data without rules.** This is the most common mistake. You must:

1. **Add rules** via `POST /2/tweets/search/stream/rules`
2. **Then connect** via `GET /2/tweets/search/stream`

Rules persist on the server -- they survive disconnections and reconnections. You don't need to re-add them after reconnecting.

#### Rule Limits

| Feature | Pay-per-use | Enterprise |
|---|---|---|
| Max rules per project | 1,000 | 25,000+ |
| Max rule length | 1,024 chars | 2,048 chars |
| Throughput | 250 posts/sec | Higher |

#### Rule Endpoints

| Method | Endpoint | Description |
|---|---|---|
| POST | `/2/tweets/search/stream/rules` | Add or delete rules |
| GET | `/2/tweets/search/stream/rules` | List current rules |

### Building Rules

Rules use the same operators as search queries but have important differences.

#### Standalone vs. Conjunction-Required Operators

**Standalone operators** can be used alone:
- Keywords: `python`, `"exact phrase"`
- Hashtags: `#typescript`
- User operators: `from:xdevelopers`, `to:user`, `@mention`
- `$cashtag`, `url:`, `context:`, `entity:`, `conversation_id:`
- Location: `place:`, `place_country:`, `point_radius:`, `bounding_box:`
- Bio operators: `bio:`, `bio_name:`, `bio_location:`

**Conjunction-required operators** MUST be combined with at least one standalone operator:
- `has:media`, `has:images`, `has:video_link`, `has:links`, `has:hashtags`, `has:mentions`, `has:geo`
- `is:retweet`, `is:reply`, `is:quote`, `is:verified`
- `lang:en`, `sample:25`

```
# INVALID - conjunction-required operator alone
has:media

# INVALID - only conjunction-required operators
has:links OR is:retweet

# VALID - standalone + conjunction-required
"X API" has:media -is:retweet
```

#### Boolean Logic

| Operator | Syntax | Example |
|---|---|---|
| AND | space | `snow day` (both required) |
| OR | `OR` | `cat OR dog` |
| NOT | `-` prefix | `cat -grumpy` |
| Grouping | `()` | `(cat OR dog) has:images -is:retweet` |

Order of operations: AND binds tighter than OR. `apple OR iphone ipad` evaluates as `apple OR (iphone ipad)`. Use parentheses to be explicit.

#### Rule Building Tips

1. **Start specific, then broaden** -- broad rules consume volume quickly
2. **Combine operators** to narrow results
3. **Use `-is:retweet`** to reduce noise (retweets are often high-volume)
4. **Use `sample:N`** (1-100) to reduce matching volume: `#python sample:25`
5. **Tag your rules** -- the `tag` field helps identify which rule matched each post
6. **Rules can be managed without disconnecting** from the stream

#### Example Rules

```
# Track a brand
(from:xdevelopers OR @XDevelopers) -is:retweet

# Topic monitoring with language filter
(AI OR "machine learning" OR "deep learning") lang:en -is:retweet has:links

# Sentiment with location
(happy OR excited) place_country:US -birthday -is:retweet

# Hashtag + media
#typescript has:images -is:retweet
```

### Connection Management

Streaming connections are long-lived HTTP requests. The server holds the connection open indefinitely, but disconnections **will** occur and must be handled.

#### Keep-Alive Heartbeat

The stream sends a blank line (`\r\n`) every **20 seconds**. If no data or heartbeat arrives in 20 seconds, the connection is dead -- trigger reconnection.

#### Why Disconnections Happen

- Server-side restarts (routine maintenance)
- Client reading too slowly (server buffer overflow)
- Network issues
- Too many concurrent connections
- Authentication errors

#### Reconnection Strategy

**On disconnect, attempt to reconnect immediately.** If that fails, use backoff:

| Error Type | Strategy | Initial Wait | Increment | Max Wait |
|---|---|---|---|---|
| TCP/IP network errors | Linear backoff | 250ms | +250ms each attempt | 16 seconds |
| HTTP errors (5xx) | Exponential backoff | 5 seconds | Double each attempt | 320 seconds |
| Rate limit (429) | Exponential backoff | 1 minute | Double each attempt | Until limit resets |

#### Architecture Best Practice

Decouple ingestion from processing with a FIFO queue:

```
Stream Connection (lightweight) -> Memory Buffer (FIFO) -> Processing Thread(s) (heavy work)
```

The stream reader thread should only read and enqueue. All parsing, database writes, and application logic happen in separate processing threads. This prevents the client from falling behind and getting disconnected.

### Data Delivery Characteristics

- **P99 latency**: ~6-7 seconds for hydrated data
- Posts are **not delivered in sorted order**
- **Duplicates may occur** -- implement deduplication
- Fields may appear in any order
- New message types may be added at any time

### Recovery After Disconnection

| Disconnection Duration | Recovery Method | Availability |
|---|---|---|
| Up to 5 minutes | `backfill_minutes` parameter on reconnect | Enterprise |
| 5 minutes to 24 hours | Recovery with `start_time`/`end_time` params | Enterprise |
| Over 24 hours | Use Search Posts endpoint to backfill | All |

For non-Enterprise users, the Search endpoint is the primary recovery mechanism for missed data during disconnections.

## HTTP Status Codes

| Code | Meaning | Action |
|---|---|---|
| 200 | Success | Process response |
| 201 | Created | Resource created successfully |
| 400 | Bad Request | Check request parameters |
| 401 | Unauthorized | Check authentication credentials |
| 403 | Forbidden | Insufficient permissions or suspended app |
| 404 | Not Found | Resource doesn't exist or is deleted |
| 429 | Too Many Requests | Rate limited - wait and retry |
| 500 | Internal Server Error | Retry with backoff |
| 503 | Service Unavailable | Retry with backoff |

## Billing

X API uses pay-per-usage billing (credit-based):
- Credits consumed per data retrieved, not per request
- 24-hour deduplication window
- Auto-recharge and spending limits configurable
- Rate limits and billing are separate concerns
