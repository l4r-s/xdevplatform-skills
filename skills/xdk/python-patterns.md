# Python XDK Patterns

Detailed reference for the `xdk` Python package. For quick start and best practices, see [SKILL.md](SKILL.md).

## Installation

```bash
pip install xdk
```

Requires Python 3.8+. Uses Pydantic models for type-safe request/response validation.

## Client Setup

```python
import os
from xdk import Client

client = Client(bearer_token=os.getenv("X_API_BEARER_TOKEN"))
```

### Client Parameters

```python
client = Client(
    base_url="https://api.x.com",       # Default API base URL
    bearer_token="YOUR_TOKEN",           # App-only auth
    access_token="OAUTH2_TOKEN",         # OAuth2 access token
    client_id="CLIENT_ID",              # For OAuth2 PKCE
    client_secret="CLIENT_SECRET",       # For OAuth2 PKCE
    redirect_uri="CALLBACK_URL",         # For OAuth2 PKCE
    token={"access_token": "...", ...},  # Full OAuth2 token dict (auto-refresh)
    scope="tweet.read users.read",       # OAuth2 scopes
    auth=oauth1_instance,               # OAuth 1.0a instance
)
```

## Authentication

### Bearer Token (App-Only)

```python
client = Client(bearer_token="YOUR_BEARER_TOKEN")
```

### OAuth 2.0 PKCE

```python
from xdk.oauth2_auth import OAuth2PKCEAuth
import webbrowser

auth = OAuth2PKCEAuth(
    client_id="YOUR_CLIENT_ID",
    redirect_uri="YOUR_CALLBACK_URL",
    scope="tweet.read users.read offline.access"
)

# Get authorization URL
auth_url = auth.get_authorization_url()
webbrowser.open(auth_url)

# After user authorizes - exchange code
callback_url = input("Paste callback URL: ")
tokens = auth.fetch_token(authorization_response=callback_url)

client = Client(bearer_token=tokens["access_token"])

# Token refresh
if auth.is_token_expired():
    tokens = auth.refresh_token()
    client = Client(bearer_token=tokens["access_token"])
```

### OAuth 1.0a

```python
from xdk import Client
from xdk.oauth1_auth import OAuth1

# With existing tokens
oauth1 = OAuth1(
    api_key="KEY", api_secret="SECRET",
    access_token="TOKEN", access_token_secret="TOKEN_SECRET",
)
client = Client(auth=oauth1)

# Full flow
oauth1 = OAuth1(
    api_key="KEY", api_secret="SECRET",
    callback="http://localhost:8080/callback",
)
request_token = oauth1.get_request_token()
auth_url = oauth1.get_authorization_url()
# User authorizes...
access_token = oauth1.get_access_token(oauth_verifier)
client = Client(auth=oauth1)
```

## Posts Client

All paginated methods return iterators. Use `for page in ...` to iterate pages.

```python
# Search recent (last 7 days)
for page in client.posts.search_recent(
    query="python -is:retweet",
    max_results=100,
    tweet_fields=["created_at", "public_metrics", "author_id"],
    expansions=["author_id"],
    user_fields=["username", "name"],
):
    for post in page.data:
        print(post.text, post.public_metrics)

# Search full-archive
for page in client.posts.search_all(
    query="from:xdevelopers",
    max_results=500,
    tweet_fields=["created_at"],
):
    for post in page.data:
        print(post.text)

# Get post by ID
response = client.posts.get_by_id(
    id="1234567890",
    tweet_fields=["text", "created_at", "public_metrics"],
    expansions=["author_id"],
    user_fields=["username"],
)
print(response.data.text)

# Get multiple posts
response = client.posts.get_by_ids(
    ids=["id1", "id2"],
    tweet_fields=["text", "created_at"],
)

# Create post
from xdk.posts.models import CreateRequest

request = CreateRequest(text="Hello from Python XDK!")
response = client.posts.create(body=request)
print(f"Created: {response.data.id}")

# Create reply
request = CreateRequest(
    text="Great post!",
    reply={"in_reply_to_tweet_id": "1234567890"},
)
response = client.posts.create(body=request)

# Create quote tweet
request = CreateRequest(
    text="So true",
    quote_tweet_id="1234567890",
)
response = client.posts.create(body=request)

# Delete post
client.posts.delete(id="1234567890")

# Get liking users
for page in client.posts.get_liking_users(id="1234567890"):
    for user in page.data:
        print(user.username)

# Get retweeted by
for page in client.posts.get_retweeted_by(id="1234567890"):
    for user in page.data:
        print(user.username)

# Get quote tweets
for page in client.posts.get_quote_tweets(id="1234567890"):
    for post in page.data:
        print(post.text)

# Analytics
response = client.posts.get_analytics(
    ids=["id1", "id2"],
    start_time="2024-01-01T00:00:00Z",
    end_time="2024-01-31T23:59:59Z",
    granularity="hour",
)

# Hide reply
client.posts.hide_reply(id="1234567890", body={"hidden": True})

# Post counts
for page in client.posts.get_counts_recent(query="query"):
    for count in page.data:
        print(count)

for page in client.posts.get_counts_all(query="query"):
    for count in page.data:
        print(count)
```

## Users Client

```python
# Get user by ID
response = client.users.get_by_id(
    id="2244994945",
    user_fields=["description", "public_metrics", "created_at"],
)
print(response.data.username)

# Get user by username
response = client.users.get_by_username(
    username="xdevelopers",
    user_fields=["description", "public_metrics"],
)

# Get multiple by IDs
response = client.users.get_by_ids(ids=["id1", "id2"])

# Get multiple by usernames
response = client.users.get_by_usernames(usernames=["user1", "user2"])

# Get authenticated user
response = client.users.get_me(user_fields=["description", "public_metrics"])

# User timeline
for page in client.users.get_posts(
    id="userId",
    max_results=100,
    tweet_fields=["created_at", "public_metrics"],
    exclude=["replies", "retweets"],
):
    for post in page.data:
        print(post.text)

# User mentions
for page in client.users.get_mentions(id="userId", max_results=100):
    for post in page.data:
        print(post.text)

# Get timeline (reverse chronological)
for page in client.users.get_timeline(id="userId", max_results=100):
    for post in page.data:
        print(post.text)

# Get followers
for page in client.users.get_followers(
    id="userId",
    max_results=100,
    user_fields=["description", "public_metrics"],
):
    for user in page.data:
        print(user.username)

# Get following
for page in client.users.get_following(id="userId", max_results=100):
    for user in page.data:
        print(user.username)

# Follow user
from xdk.users.models import FollowRequest
client.users.follow_user(id="myUserId", body=FollowRequest(target_user_id="targetId"))

# Unfollow user
client.users.unfollow_user(source_user_id="myUserId", target_user_id="targetId")

# Block user
client.users.block_user(id="myUserId", body={"target_user_id": "targetId"})

# Mute user
client.users.mute_user(id="myUserId", body={"target_user_id": "targetId"})

# Like a post
client.users.like_post(id="myUserId", body={"tweet_id": "postId"})

# Unlike
client.users.unlike_post(id="myUserId", tweet_id="postId")

# Retweet
client.users.retweet(id="myUserId", body={"tweet_id": "postId"})

# Undo retweet
client.users.undo_retweet(id="myUserId", tweet_id="postId")

# Liked posts
for page in client.users.get_liked_posts(id="userId"):
    for post in page.data:
        print(post.text)

# Bookmarks
for page in client.users.get_bookmarks(id="userId"):
    for post in page.data:
        print(post.text)

# Add bookmark
client.users.add_bookmark(id="userId", body={"tweet_id": "postId"})

# Remove bookmark
client.users.remove_bookmark(id="userId", tweet_id="postId")

# Search users
for page in client.users.search(
    query="developer",
    max_results=100,
    user_fields=["description", "public_metrics"],
):
    for user in page.data:
        print(user.username)

# Reposts of me
for page in client.users.get_reposts_of_me():
    for post in page.data:
        print(post.text)
```

## Streaming

**IMPORTANT:** The filtered stream (`client.stream.posts()`) delivers NO data without rules. You must add rules first. The sampled stream (`client.stream.posts_sample()`) requires no rules.

### Filtered Stream - Complete Workflow

```python
from xdk.stream.models import UpdateRulesRequest

# Step 1: ALWAYS add rules before connecting to filtered stream
# Rules persist on the server - they survive disconnections
add_request = UpdateRulesRequest(**{
    "add": [
        {"value": "from:xdevelopers -is:retweet", "tag": "official"},
        {"value": "#python has:media -is:retweet", "tag": "python-media"},
    ]
})
client.stream.update_rules(body=add_request)

# Step 2: Verify rules are active
for page in client.stream.get_rules():
    if page.data:
        for rule in page.data:
            print(f"Active rule: {rule.id} -> {rule.value} (tag: {rule.tag})")
    break

# Step 3: Connect and consume
for post_response in client.stream.posts(
    tweet_fields=["id", "text", "created_at"],
    expansions=["author_id"],
    user_fields=["username"],
):
    data = post_response.model_dump()
    if "data" in data and data["data"]:
        tweet = data["data"]
        print(f"Post: {tweet.get('text', '')}")
```

### Rule Management

```python
from xdk.stream.models import UpdateRulesRequest

# Add rules (can be done while stream is connected from another process)
add_request = UpdateRulesRequest(**{
    "add": [
        {"value": "(AI OR \"machine learning\") lang:en -is:retweet", "tag": "ai-en"},
        {"value": "from:xdevelopers OR @XDevelopers", "tag": "brand"},
    ]
})
client.stream.update_rules(body=add_request)

# List current rules
for page in client.stream.get_rules():
    if page.data:
        for rule in page.data:
            print(f"ID: {rule.id}, Value: {rule.value}, Tag: {rule.tag}")
    break

# Delete specific rules by ID
delete_request = UpdateRulesRequest(**{
    "delete": {"ids": ["rule-id-1", "rule-id-2"]}
})
client.stream.update_rules(body=delete_request)

# Replace all rules: delete existing, then add new
for page in client.stream.get_rules():
    if page.data:
        ids = [rule.id for rule in page.data]
        if ids:
            client.stream.update_rules(body=UpdateRulesRequest(**{
                "delete": {"ids": ids}
            }))
    break

client.stream.update_rules(body=UpdateRulesRequest(**{
    "add": [{"value": "new rule here", "tag": "fresh-start"}]
}))
```

### Reconnection Pattern

```python
import time
from xdk import Client

def connect_with_reconnect(client: Client):
    """Connect to filtered stream with automatic reconnection."""
    retry_delay = 0
    max_delay = 320  # seconds

    while True:
        try:
            if retry_delay > 0:
                print(f"Reconnecting in {retry_delay}s...")
                time.sleep(retry_delay)

            for post_response in client.stream.posts(
                tweet_fields=["id", "text", "created_at"],
                expansions=["author_id"],
            ):
                retry_delay = 0  # Reset on successful data
                data = post_response.model_dump()
                if "data" in data and data["data"]:
                    # Hand off to processing queue - don't do heavy work here
                    process_queue.put(data["data"])

            # Stream ended normally - reconnect immediately
            retry_delay = 0

        except Exception as e:
            # Exponential backoff for errors
            retry_delay = 5 if retry_delay == 0 else min(retry_delay * 2, max_delay)
            print(f"Stream error: {e}, backing off {retry_delay}s")
```

### Async Streaming (with threading)

```python
import asyncio
from asyncio import Queue
import threading
from xdk import Client

async def stream_posts_async(client: Client):
    """Run the synchronous stream in a background thread, consume async."""
    queue = Queue()
    loop = asyncio.get_event_loop()
    stop = threading.Event()

    def run_stream():
        for post in client.stream.posts():
            if stop.is_set():
                break
            asyncio.run_coroutine_threadsafe(queue.put(post), loop)
        asyncio.run_coroutine_threadsafe(queue.put(None), loop)

    threading.Thread(target=run_stream, daemon=True).start()

    while True:
        post = await queue.get()
        if post is None:
            break
        data = post.model_dump()
        if "data" in data and data["data"]:
            print(f"Post: {data['data'].get('text', '')}")

    stop.set()

asyncio.run(stream_posts_async(client))
```

### Stream Configuration (reconnection/backoff)

```python
from xdk.streaming import StreamConfig

config = StreamConfig(
    max_retries=10,              # Max reconnection attempts
    initial_backoff=1.0,         # First retry wait (seconds)
    max_backoff=64.0,            # Max wait between retries
    backoff_multiplier=2.0,      # Exponential factor
    jitter=True,                 # Add randomness to avoid thundering herd
    timeout=None,                # Connection timeout (None = no timeout)
    chunk_size=1024,             # Read buffer size
    on_connect=lambda: print("Connected to stream"),
    on_disconnect=lambda e: print(f"Disconnected: {e}"),
    on_reconnect=lambda retry, backoff: print(f"Reconnecting (attempt {retry}, wait {backoff}s)"),
    on_error=lambda error: print(f"Stream error: {error}"),
)

# Apply config to any stream
for post in client.stream.posts(stream_config=config):
    data = post.model_dump()
    if "data" in data and data["data"]:
        print(data["data"].get("text", ""))
```

### Sampled Stream (no rules needed)

```python
# 1% random sample of all public posts - no rules required
for post in client.stream.posts_sample(
    tweet_fields=["id", "text", "created_at"],
    expansions=["author_id"],
    user_fields=["username"],
):
    data = post.model_dump()
    if "data" in data and data["data"]:
        print(data["data"].get("text", ""))
```

### Other Stream Types

```python
# 10% sample (Enterprise)
for post in client.stream.posts_sample10():
    pass

# Firehose - all posts (Enterprise)
for post in client.stream.posts_firehose():
    pass

# Language-specific firehose (Enterprise)
for post in client.stream.posts_firehose_lang_en():
    pass
for post in client.stream.posts_firehose_lang_ja():
    pass
for post in client.stream.posts_firehose_lang_ko():
    pass
for post in client.stream.posts_firehose_lang_pt():
    pass

# Compliance streams (Enterprise)
for event in client.stream.posts_compliance_stream():
    pass
for event in client.stream.users_compliance_stream():
    pass
for event in client.stream.likes_compliance_stream():
    pass
```

## Pagination

### Automatic (Recommended)

All list endpoints return iterators that handle `next_token` automatically:

```python
# Collects all results across pages
all_posts = []
for page in client.posts.search_recent(query="python", max_results=100):
    if page.data:
        all_posts.extend(page.data)
    print(f"Fetched {len(page.data)} posts (total: {len(all_posts)})")
```

### Manual Pagination

```python
# Get first page only
first_page = next(client.posts.search_recent(
    query="xdk python sdk",
    max_results=100,
))

# Extract next_token
next_token = None
if hasattr(first_page, "meta") and first_page.meta:
    next_token = getattr(first_page.meta, "next_token", None)

# Fetch next page
if next_token:
    second_page = next(client.posts.search_recent(
        query="xdk python sdk",
        max_results=100,
        pagination_token=next_token,
    ))
```

### Cursor Helper

```python
from xdk.paginator import cursor

posts_cursor = cursor(client.posts.search_recent)(query="python", max_results=100)

# Iterate pages (with limit)
for page in posts_cursor.pages(limit=10):
    print(f"Page: {len(page.data)} posts")

# Iterate individual items (with limit)
for post in posts_cursor.items(limit=1000):
    print(post.text)
```

## Media

```python
# Upload (one-shot)
response = client.media.upload(media=file_buffer, media_type="image/jpeg")

# Chunked upload
init = client.media.initialize_upload(total_bytes=file_size, media_type="video/mp4")
media_id = init.media_id

client.media.append_upload(id=media_id, media=chunk1, segment_index=0)
client.media.append_upload(id=media_id, media=chunk2, segment_index=1)
result = client.media.finalize_upload(id=media_id)

# Check status
status = client.media.get_upload_status(id=media_id)

# Get media by key
media = client.media.get_by_key(
    media_key="3_1234567890",
    media_fields=["url", "preview_image_url", "type"],
)

# Create metadata (alt text)
client.media.create_metadata(body={
    "media_id": media_id,
    "alt_text": {"text": "Image description"},
})

# Create subtitles
client.media.create_subtitles(body={
    "media_id": media_id,
    "media_category": "tweet_video",
    "subtitle_info": {
        "subtitles": [
            {"language_code": "en", "display_name": "English", "media_id": subtitle_id}
        ]
    },
})

# Analytics
response = client.media.get_analytics(media_keys=["key1", "key2"])
```

## Lists

```python
from xdk.lists.models import CreateRequest, UpdateRequest

# Create list
response = client.lists.create(body=CreateRequest(
    name="My List", description="Curated list", private=False,
))

# Update list
client.lists.update(id="listId", body=UpdateRequest(name="Updated"))

# Delete list
client.lists.delete(id="listId")

# Get list
response = client.lists.get_by_id(id="listId")

# Get members
for page in client.lists.get_members(id="listId"):
    for user in page.data:
        print(user.username)

# Get list tweets
for page in client.lists.get_tweets(id="listId"):
    for post in page.data:
        print(post.text)

# Add/remove members
client.lists.add_member(id="listId", body={"user_id": "userId"})
client.lists.remove_member(id="listId", user_id="userId")
```

## Direct Messages

```python
# Create conversation
response = client.direct_messages.create_conversation(body={
    "conversation_type": "Group",
    "participant_ids": ["userId1", "userId2"],
    "message": {"text": "Hello everyone!"},
})

# Send by conversation ID
client.direct_messages.create_by_conversation_id(
    dm_conversation_id="convId",
    body={"text": "Follow-up"},
)

# Send by participant ID (1-on-1)
client.direct_messages.create_by_participant_id(
    participant_id="userId",
    body={"text": "Hey!"},
)

# Get events
for page in client.direct_messages.get_events_by_conversation_id(
    dm_conversation_id="convId",
    max_results=50,
):
    for event in page.data:
        print(event)

# Delete event
client.direct_messages.delete_event(id="eventId")
```

## Other Clients

```python
# Spaces
response = client.spaces.get_by_id(id="spaceId")
response = client.spaces.search(query="topic")

# Communities
response = client.communities.get_by_id(id="communityId")
response = client.communities.search(query="topic")

# Community Notes
client.community_notes.create(body={"text": "Note", "tweet_id": "postId"})

# Trends
response = client.trends.get_by_woeid(woeid=1)  # 1 = worldwide
response = client.trends.get_personalized()

# Usage
response = client.usage.get_tweets_usage()

# Compliance
response = client.compliance.create_job(body={"type": "tweets", "name": "job"})
for page in client.compliance.get_jobs(type="tweets"):
    for job in page.data:
        print(job)

# Webhooks
response = client.webhooks.create(body={"url": "https://example.com/webhook"})

# News
for page in client.news.search(query="topic"):
    for article in page.data:
        print(article)
```

## Pydantic Model Access

Response models support both attribute and dict access:

```python
post = response.data

# Attribute access (preferred)
text = post.text
author_id = post.author_id

# Dict access (for compatibility)
data_dict = post.model_dump()
text = data_dict.get("text", "")
```

## Error Handling

```python
try:
    response = client.posts.get_by_id(id="123")
except Exception as e:
    print(f"Error: {e}")
    # Common errors:
    # 401 - Authentication failed
    # 403 - Forbidden (insufficient permissions)
    # 404 - Not found
    # 429 - Rate limited
```

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| 401 Unauthorized | Invalid/expired token | Regenerate bearer token or refresh OAuth2 token |
| 403 Forbidden | Insufficient app permissions | Check app permissions in Developer Console |
| 404 Not Found | Deleted/nonexistent resource | Verify the ID exists |
| 420 Enhance Your Calm | Stream rate limited | Wait and retry with backoff |
| 429 Too Many Requests | Rate limited | Wait for `x-rate-limit-reset`, implement backoff |

## Full API Reference

For complete method signatures and parameters, see:
- [Python SDK reference](https://github.com/xdevplatform/docs/tree/main/xdks/python/reference) - All module and client docs
- [Client classes](https://github.com/xdevplatform/docs/tree/main/xdks/python/reference) - Files matching `xdk.*.client.mdx`
- [Request/response models](https://github.com/xdevplatform/docs/tree/main/xdks/python/reference) - Files matching `xdk.*.models.mdx`
