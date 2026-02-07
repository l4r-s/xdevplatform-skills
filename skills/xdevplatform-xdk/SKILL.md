---
name: xdk
description: Build applications with the X API using the official TypeScript and Python XDKs. Provides SDK setup, authentication, pagination, streaming, and best practices. Use when the user mentions "X API", "XDK", "Twitter API", "post", "tweet", "stream", "filtered stream", "TypeScript SDK", "Python SDK", "@xdevplatform/xdk", or wants to interact with the X platform programmatically.
---

# XDK - X Developer Kit

Official TypeScript and Python SDKs for the X API v2. Both SDKs are generated from the same OpenAPI spec and share identical API coverage.

| | TypeScript | Python |
|---|---|---|
| **Package** | `@xdevplatform/xdk` | `xdk` |
| **Install** | `npm install @xdevplatform/xdk` | `pip install xdk` |
| **Runtime** | Node.js 16+, TypeScript 4.5+ | Python 3.8+ |
| **Repo** | [github.com/xdevplatform/xdk](https://github.com/xdevplatform/xdk) | [github.com/xdevplatform/xdk](https://github.com/xdevplatform/xdk) |
| **Samples** | [samples/javascript](https://github.com/xdevplatform/samples/tree/main/javascript) | [samples/python](https://github.com/xdevplatform/samples/tree/main/python) |

## Quick Start

### TypeScript

```typescript
import { Client, type ClientConfig } from '@xdevplatform/xdk';

const client = new Client({ bearerToken: process.env.X_API_BEARER_TOKEN });

const response = await client.users.getByUsername('XDevelopers', {
  userFields: ['description', 'public_metrics'],
});
console.log(response.data?.username);
```

### Python

```python
import os
from xdk import Client

client = Client(bearer_token=os.getenv("X_API_BEARER_TOKEN"))

for page in client.posts.search_recent(query="python", max_results=10):
    if page.data:
        print(page.data[0].text)
        break
```

## Client Architecture

Both SDKs expose a main `Client` with specialized sub-clients:

| Sub-client | Purpose |
|---|---|
| `client.posts` | Search, create, delete, lookup, analytics, likes, reposts, quote tweets |
| `client.users` | Lookup, follow/unfollow, block/mute, timelines, mentions |
| `client.stream` | Filtered stream, sampled stream, firehose, rules management |
| `client.media` | Upload (chunked), metadata, subtitles, analytics |
| `client.lists` | Create, manage members, get list tweets |
| `client.directMessages` | Send/receive DMs, manage conversations |
| `client.spaces` | Lookup, search, get posts from Spaces |
| `client.communities` | Search, get community by ID |
| `client.communityNotes` | Create, evaluate, search notes |
| `client.trends` | By WOEID, personalized, AI trends |
| `client.webhooks` | Create, manage, replay webhooks |
| `client.compliance` | Compliance jobs |
| `client.usage` | API usage stats |
| `client.activity` | Activity subscriptions and streaming |
| `client.accountActivity` | Account activity API |
| `client.connections` | Connection management |
| `client.news` | News search and lookup |

## Authentication

Three methods, both SDKs:

### 1. Bearer Token (App-Only) - Read-only public data

```typescript
// TypeScript
const client = new Client({ bearerToken: 'YOUR_TOKEN' });
```

```python
# Python
client = Client(bearer_token="YOUR_TOKEN")
```

### 2. OAuth 2.0 PKCE - User-context operations (recommended)

```typescript
// TypeScript
import { Client, OAuth2, generateCodeVerifier, generateCodeChallenge } from '@xdevplatform/xdk';

const oauth2 = new OAuth2({
  clientId: 'your-client-id',
  clientSecret: 'your-client-secret',
  redirectUri: 'https://example.com/callback',
  scope: ['tweet.read', 'users.read', 'offline.access'],
});

const codeVerifier = generateCodeVerifier();
const codeChallenge = await generateCodeChallenge(codeVerifier);
oauth2.setPkceParameters(codeVerifier, codeChallenge);

const authUrl = await oauth2.getAuthorizationUrl('state');
// User authorizes -> callback returns code
const tokens = await oauth2.exchangeCode(authCode, codeVerifier);
const client = new Client({ accessToken: tokens.access_token });
```

```python
# Python
from xdk.oauth2_auth import OAuth2PKCEAuth

auth = OAuth2PKCEAuth(
    client_id="YOUR_CLIENT_ID",
    redirect_uri="YOUR_CALLBACK_URL",
    scope="tweet.read users.read offline.access"
)
auth_url = auth.get_authorization_url()
# User authorizes -> callback returns URL
tokens = auth.fetch_token(authorization_response=callback_url)
client = Client(bearer_token=tokens["access_token"])
```

### 3. OAuth 1.0a - Legacy user context

```typescript
// TypeScript
import { Client, OAuth1 } from '@xdevplatform/xdk';

const oauth1 = new OAuth1({
  apiKey: 'key', apiSecret: 'secret',
  accessToken: 'token', accessTokenSecret: 'token-secret',
});
const client = new Client({ oauth1 });
```

```python
# Python
from xdk import Client
from xdk.oauth1_auth import OAuth1

oauth1 = OAuth1(
    api_key="key", api_secret="secret",
    access_token="token", access_token_secret="token-secret",
)
client = Client(auth=oauth1)
```

## Fields and Expansions

The X API returns minimal data by default (`id`, `text`). Always request the fields you need.

```typescript
// TypeScript - get post with author info
const post = await client.posts.getById('123456', {
  tweetFields: ['created_at', 'public_metrics', 'author_id'],
  expansions: ['author_id', 'attachments.media_keys'],
  userFields: ['username', 'name', 'profile_image_url'],
  mediaFields: ['url', 'preview_image_url', 'type'],
});
// post.data = the post, post.includes.users = expanded author
```

```python
# Python - search with fields
for page in client.posts.search_recent(
    query="python",
    max_results=100,
    tweet_fields=["created_at", "public_metrics", "author_id"],
    expansions=["author_id"],
    user_fields=["username", "name"],
):
    for post in page.data:
        print(post.text, post.public_metrics)
```

Common expansions: `author_id`, `referenced_tweets.id`, `attachments.media_keys`, `in_reply_to_user_id`, `geo.place_id`.

## Pagination

### TypeScript - Paginator wrapper with async iteration

```typescript
import { Client, UserPaginator, PaginatedResponse, Schemas } from '@xdevplatform/xdk';

const client = new Client({ bearerToken: 'token' });

const followers = new UserPaginator(
  async (token?: string): Promise<PaginatedResponse<Schemas.User>> => {
    const res = await client.users.getFollowers('userId', {
      maxResults: 100,
      paginationToken: token,
      userFields: ['id', 'name', 'username'],
    });
    return { data: res.data ?? [], meta: res.meta, includes: res.includes, errors: res.errors };
  }
);

// Async iteration (recommended)
for await (const user of followers) {
  console.log(user.username);
}

// Or manual paging
await followers.fetchNext();
while (!followers.done) {
  await followers.fetchNext();
}
console.log(followers.users.length);
```

### Python - Iterator-based (automatic)

```python
# Automatic - SDK handles next_token
all_posts = []
for page in client.posts.search_recent(query="python", max_results=100):
    all_posts.extend(page.data)

# Manual - extract next_token yourself
first_page = next(client.posts.search_recent(query="xdk", max_results=100))
next_token = first_page.meta.next_token if first_page.meta else None
```

## Streaming

The X API offers several real-time streaming endpoints. Choose based on your use case:

| Stream Type | Method | Rules Required | Volume | Access |
|---|---|---|---|---|
| **Filtered Stream** | `client.stream.posts()` | **Yes - must add rules first** | Only matching posts | All |
| **Sampled Stream (1%)** | `client.stream.postsSample()` | No | ~1% random sample | All |
| **Sampled Stream (10%)** | `client.stream.postsSample10()` | No | ~10% random sample | Enterprise |
| **Firehose** | `client.stream.postsFirehose()` | No | All posts | Enterprise |

**IMPORTANT: The filtered stream delivers NOTHING without rules.** You must add at least one rule before connecting. Connecting without rules produces a silent empty stream with no error.

### Filtered Stream Workflow

The correct order is always: **add rules -> connect -> process posts**.

```typescript
// TypeScript
// Step 1: Add rules BEFORE connecting
await client.stream.updateRules({
  add: [{ value: 'from:xdevelopers -is:retweet', tag: 'official' }],
});

// Step 2: Connect to stream
const stream = await client.stream.posts({
  tweetFields: ['id', 'text', 'created_at'],
  expansions: ['author_id'],
  userFields: ['username'],
});

// Step 3: Consume events
stream.on('data', (event) => {
  console.log(event.data?.text);
  console.log('Matched rules:', event.matching_rules);
});
stream.on('error', (e) => console.error(e));

// Or use async iteration
for await (const event of stream) {
  console.log(event);
}

// Always close when done
stream.close();
```

```python
# Python
from xdk.stream.models import UpdateRulesRequest

# Step 1: Add rules BEFORE connecting
request = UpdateRulesRequest(**{
    "add": [{"value": "from:xdevelopers -is:retweet", "tag": "official"}]
})
client.stream.update_rules(body=request)

# Step 2: Connect and consume
for post_response in client.stream.posts():
    data = post_response.model_dump()
    if 'data' in data and data['data']:
        print(data['data'].get('text', ''))
```

### Sampled Stream (no rules needed)

```typescript
// TypeScript - 1% random sample of all posts
const stream = await client.stream.postsSample({
  tweetFields: ['id', 'text', 'created_at'],
});
for await (const event of stream) {
  console.log(event);
}
```

```python
# Python - 1% random sample
for post in client.stream.posts_sample():
    data = post.model_dump()
    if 'data' in data and data['data']:
        print(data['data'].get('text', ''))
```

### Connection Management

- Streams send a **keep-alive heartbeat every 20 seconds** -- if nothing arrives in 20s, reconnect
- **Disconnections are expected** (server restarts, network issues, buffer overflow) -- always implement reconnection logic
- Only **1 concurrent connection** allowed for filtered stream (pay-per-use)
- You can **add/remove rules without disconnecting** from the stream
- **Decouple ingestion from processing** -- use a FIFO queue between the stream reader and your processing logic

For detailed streaming patterns, reconnection strategies, and rule-building guidance, see [api-concepts.md](api-concepts.md).

## Best Practices

1. **Always specify fields** - Never rely on defaults. Request exactly what you need to minimize payload and billing.
2. **Use expansions** - Get related objects (author, media) in one call instead of multiple requests.
3. **Environment variables** - Never hardcode tokens. Use `process.env` (TS) or `os.getenv` (Python).
4. **Handle rate limits** - Check `x-rate-limit-remaining` headers. Use backoff on 429 errors. The Python SDK handles rate limit backoff automatically for pagination.
5. **Use streaming over polling** - For real-time data, use filtered stream instead of repeated search calls.
6. **Close streams** - Always close stream connections when done to avoid resource leaks.
7. **Paginate large results** - Use async iteration (TS) or iterator (Python) for automatic pagination.
8. **Use max_results** - Set `max_results` to the maximum allowed to minimize API calls.
9. **Cache responses** - Store results locally to reduce repeated requests.
10. **Use TypeScript types** - Import `Schemas.*` types for compile-time safety.

## Error Handling

```typescript
// TypeScript
import { ApiError } from '@xdevplatform/xdk';

try {
  const post = await client.posts.getById('123');
} catch (error) {
  if (error instanceof ApiError) {
    console.error(`API Error ${error.statusCode}: ${error.message}`);
  }
}
```

```python
# Python
try:
    response = client.posts.get_by_id(id="123")
except Exception as e:
    print(f"Error: {e}")
```

## Additional Resources

- For detailed TypeScript patterns, see [typescript-patterns.md](typescript-patterns.md)
- For detailed Python patterns, see [python-patterns.md](python-patterns.md)
- For core X API concepts (response structure, rate limits, search operators), see [api-concepts.md](api-concepts.md)
- Full X API docs: [xdevplatform/docs/x-api](https://github.com/xdevplatform/docs/tree/main/x-api)
- Full SDK reference docs: [xdevplatform/docs/xdks](https://github.com/xdevplatform/docs/tree/main/xdks)
- XDK source: [xdevplatform/xdk](https://github.com/xdevplatform/xdk)
- Code samples: [xdevplatform/samples](https://github.com/xdevplatform/samples)
