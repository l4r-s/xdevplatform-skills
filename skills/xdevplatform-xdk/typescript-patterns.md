# TypeScript XDK Patterns

Detailed reference for `@xdevplatform/xdk`. For quick start and best practices, see [SKILL.md](SKILL.md).

## Installation

```bash
npm install @xdevplatform/xdk
# or: yarn add @xdevplatform/xdk
# or: pnpm add @xdevplatform/xdk
```

Requires Node.js 16+, TypeScript 4.5+. Full type definitions included.

## Client Setup

```typescript
import {
  Client,
  type ClientConfig,
  type Schemas,
  type Users,
  type Posts,
} from '@xdevplatform/xdk';

const client = new Client({ bearerToken: process.env.X_API_BEARER_TOKEN });
```

### Available Types

Import response types from namespaced modules:

```typescript
import { type Users, type Posts, type Schemas } from '@xdevplatform/xdk';

// Use as return types
const response: Users.GetByUsernameResponse = await client.users.getByUsername('XDevelopers');
const post: Schemas.Tweet = response.data;
```

## Posts Client

```typescript
// Lookup by ID
const post = await client.posts.getById('1234567890', {
  tweetFields: ['id', 'text', 'created_at', 'public_metrics', 'author_id'],
  expansions: ['author_id'],
  userFields: ['username', 'name'],
});

// Lookup multiple
const posts = await client.posts.getByIds(['id1', 'id2'], {
  tweetFields: ['text', 'created_at'],
});

// Search recent (last 7 days)
const results = await client.posts.searchRecent('typescript -is:retweet', {
  maxResults: 100,
  tweetFields: ['created_at', 'public_metrics'],
  expansions: ['author_id'],
  userFields: ['username'],
});

// Search full-archive (all time)
const archive = await client.posts.searchAll('from:xdevelopers', {
  maxResults: 500,
  tweetFields: ['created_at'],
});

// Create post
const newPost = await client.posts.create({ text: 'Hello from the XDK!' });

// Create post with media
const mediaPost = await client.posts.create({
  text: 'Check this out!',
  media: { media_ids: ['media_id_here'] },
});

// Create reply
const reply = await client.posts.create({
  text: 'Great post!',
  reply: { in_reply_to_tweet_id: '1234567890' },
});

// Create quote
const quote = await client.posts.create({
  text: 'So true',
  quote_tweet_id: '1234567890',
});

// Delete post
await client.posts.delete('1234567890');

// Get liking users
const likers = await client.posts.getLikingUsers('1234567890', {
  userFields: ['username', 'public_metrics'],
});

// Get reposted by
const reposters = await client.posts.getRetweetedBy('1234567890');

// Get quote tweets
const quotes = await client.posts.getQuoteTweets('1234567890');

// Get analytics
const analytics = await client.posts.getAnalytics(
  ['id1', 'id2'],
  '2024-01-01T00:00:00Z',
  '2024-01-31T23:59:59Z',
  'hour',
);

// Hide reply
await client.posts.hideReply('1234567890', { hidden: true });

// Post counts (recent)
const counts = await client.posts.getCountsRecent('query here');

// Post counts (all)
const allCounts = await client.posts.getCountsAll('query here');
```

## Users Client

```typescript
// Lookup by ID
const user = await client.users.getById('2244994945', {
  userFields: ['description', 'public_metrics', 'created_at'],
});

// Lookup by username
const user = await client.users.getByUsername('xdevelopers', {
  userFields: ['description', 'public_metrics'],
});

// Lookup multiple by IDs
const users = await client.users.getByIds(['id1', 'id2']);

// Lookup multiple by usernames
const users = await client.users.getByUsernames(['user1', 'user2']);

// Get authenticated user
const me = await client.users.getMe({
  userFields: ['description', 'public_metrics'],
});

// Get user's posts (timeline)
const timeline = await client.users.getPosts('userId', {
  maxResults: 100,
  tweetFields: ['created_at', 'public_metrics'],
  exclude: ['replies', 'retweets'],
});

// Get user's mentions
const mentions = await client.users.getMentions('userId', {
  maxResults: 100,
  tweetFields: ['created_at'],
});

// Get reverse chronological timeline
const home = await client.users.getReverseChronological('userId', {
  maxResults: 100,
});

// Get followers
const followers = await client.users.getFollowers('userId', {
  maxResults: 100,
  userFields: ['description', 'public_metrics'],
});

// Get following
const following = await client.users.getFollowing('userId', {
  maxResults: 100,
});

// Follow user
await client.users.followUser('myUserId', { target_user_id: 'targetUserId' });

// Unfollow user
await client.users.unfollowUser('myUserId', 'targetUserId');

// Block user
await client.users.blockUser('myUserId', { target_user_id: 'targetUserId' });

// Mute user
await client.users.muteUser('myUserId', { target_user_id: 'targetUserId' });

// Like a post
await client.users.likePost('myUserId', { tweet_id: 'postId' });

// Unlike a post
await client.users.unlikePost('myUserId', 'postId');

// Retweet
await client.users.retweet('myUserId', { tweet_id: 'postId' });

// Undo retweet
await client.users.undoRetweet('myUserId', 'postId');

// Get liked posts
const liked = await client.users.getLikedPosts('userId');

// Get bookmarks
const bookmarks = await client.users.getBookmarks('userId');

// Add bookmark
await client.users.addBookmark('userId', { tweet_id: 'postId' });

// Remove bookmark
await client.users.removeBookmark('userId', 'postId');

// Search users
const userSearch = await client.users.search('query', {
  maxResults: 100,
  userFields: ['description', 'public_metrics'],
});
```

## Streaming

**IMPORTANT:** The filtered stream (`client.stream.posts()`) delivers NO data without rules. You must add rules first. The sampled stream (`client.stream.postsSample()`) requires no rules.

### Filtered Stream - Complete Workflow

```typescript
// Step 1: ALWAYS add rules before connecting to filtered stream
// Rules persist on the server - they survive disconnections
const rulesResponse = await client.stream.updateRules({
  add: [
    { value: 'from:xdevelopers -is:retweet', tag: 'official' },
    { value: '#typescript has:media -is:retweet', tag: 'ts-media' },
  ],
});

// Step 2: Verify rules are active
const rules = await client.stream.getRules();
console.log('Active rules:', rules);

// Step 3: Connect to the stream
const stream = await client.stream.posts({
  tweetFields: ['id', 'text', 'created_at', 'public_metrics'],
  expansions: ['author_id'],
  userFields: ['username', 'name'],
});

// Step 4: Consume events (event-driven approach)
stream.on('data', (event) => {
  console.log('Post:', event.data?.text);
  console.log('Matched rules:', event.matching_rules);
});
stream.on('error', (e) => console.error('Stream error:', e));
stream.on('keepAlive', () => {
  // Heartbeat received - connection is alive
  // These arrive every ~20 seconds. If none arrive in 20s, reconnect.
});
stream.on('close', () => {
  console.log('Stream closed - implement reconnection here');
  // See reconnection strategy below
});

// Or async iteration approach
for await (const event of stream) {
  console.log(event.data?.text);
}

// Always close when done to release resources
stream.close();
```

### Rule Management

```typescript
// Add rules (can be done while stream is connected)
await client.stream.updateRules({
  add: [
    { value: '(AI OR "machine learning") lang:en -is:retweet', tag: 'ai-en' },
    { value: 'from:xdevelopers OR @XDevelopers', tag: 'brand' },
  ],
});

// List current rules
const rules = await client.stream.getRules();

// Delete specific rules by ID
await client.stream.updateRules({
  delete: { ids: ['rule-id-1', 'rule-id-2'] },
});

// Replace all rules: delete existing, then add new
// Useful for a clean slate
const existingRules = await client.stream.getRules();
if (existingRules.data?.length) {
  await client.stream.updateRules({
    delete: { ids: existingRules.data.map((r) => r.id) },
  });
}
await client.stream.updateRules({
  add: [{ value: 'new rule here', tag: 'fresh-start' }],
});
```

### Reconnection Pattern

```typescript
async function connectWithReconnect(client: Client) {
  let retryDelay = 0;

  while (true) {
    try {
      if (retryDelay > 0) {
        console.log(`Reconnecting in ${retryDelay}ms...`);
        await new Promise((r) => setTimeout(r, retryDelay));
      }

      const stream = await client.stream.posts({
        tweetFields: ['id', 'text', 'created_at'],
        expansions: ['author_id'],
      });

      retryDelay = 0; // Reset on successful connection

      stream.on('data', (event) => {
        // Hand off to processing queue - don't do heavy work here
        processQueue.push(event);
      });

      stream.on('error', (e) => {
        console.error('Stream error:', e);
      });

      // Wait for stream to close (blocks until disconnection)
      await new Promise<void>((resolve) => {
        stream.on('close', () => resolve());
      });

      // Connection dropped - retry immediately first
      retryDelay = 0;
    } catch (error: unknown) {
      // Exponential backoff for HTTP errors
      retryDelay = retryDelay === 0 ? 5000 : Math.min(retryDelay * 2, 320000);
      console.error('Connection failed, backing off:', error);
    }
  }
}
```

### Sampled Stream (no rules needed)

```typescript
// 1% random sample of all public posts - no rules required
const stream = await client.stream.postsSample({
  tweetFields: ['id', 'text', 'created_at'],
  expansions: ['author_id'],
  userFields: ['username'],
});

for await (const event of stream) {
  console.log(event);
}
```

### Other Stream Types

```typescript
// 10% sample (Enterprise)
const sample10 = await client.stream.postsSample10(params);

// Firehose - all posts (Enterprise)
const firehose = await client.stream.postsFirehose(params);

// Language-specific firehose (Enterprise)
const enStream = await client.stream.postsFirehoseLangEn(params);
const jaStream = await client.stream.postsFirehoseLangJa(params);
const koStream = await client.stream.postsFirehoseLangKo(params);
const ptStream = await client.stream.postsFirehoseLangPt(params);

// Compliance streams (Enterprise)
const tweetCompliance = await client.stream.postsComplianceStream(params);
const userCompliance = await client.stream.usersComplianceStream(params);
const likeCompliance = await client.stream.likesComplianceStream(params);
```

## Pagination

The SDK provides typed paginators: `PostPaginator`, `UserPaginator`, `EventPaginator`.

```typescript
import {
  Client,
  UserPaginator,
  PostPaginator,
  PaginatedResponse,
  Schemas,
} from '@xdevplatform/xdk';

// Wrap any paginated endpoint
const followers = new UserPaginator(
  async (token?: string): Promise<PaginatedResponse<Schemas.User>> => {
    const res = await client.users.getFollowers('userId', {
      maxResults: 100,
      paginationToken: token,
      userFields: ['id', 'name', 'username'],
    });
    return {
      data: res.data ?? [],
      meta: res.meta,
      includes: res.includes,
      errors: res.errors,
    };
  }
);

// Async iteration - yields individual items
for await (const user of followers) {
  console.log(user.username);
}

// Manual page-by-page
await followers.fetchNext(); // first page
console.log(followers.users); // items so far
console.log(followers.meta?.nextToken); // next page token
console.log(followers.done); // boolean

while (!followers.done) {
  await followers.fetchNext();
}

// Get next page as new independent paginator
await followers.fetchNext();
if (!followers.done) {
  const page2 = await followers.next();
  await page2.fetchNext();
}

// Rate limit detection
try {
  for await (const user of followers) {
    // process
  }
} catch (err) {
  if (followers.rateLimited) {
    console.error('Rate limited - implement backoff');
  }
}
```

### PostPaginator Example

```typescript
const searchResults = new PostPaginator(
  async (token?: string): Promise<PaginatedResponse<Schemas.Tweet>> => {
    const res = await client.posts.searchRecent('typescript', {
      maxResults: 100,
      paginationToken: token,
      tweetFields: ['created_at', 'public_metrics'],
    });
    return { data: res.data ?? [], meta: res.meta, includes: res.includes, errors: res.errors };
  }
);

for await (const post of searchResults) {
  console.log(post.text, post.public_metrics);
}
```

## Media Upload (OAuth1 Workaround)

**IMPORTANT:** The TypeScript XDK has a bug with OAuth1 authentication where the body stream is consumed during signature computation, causing `client.media.upload()` and `client.media.appendUpload()` to fail with `Body is unusable: Body has already been read`. Use the hybrid approach below.

The workaround uses the XDK for all steps that work (INIT, FINALIZE, POST) and a direct `fetch` with manual OAuth1 header signing for the APPEND step only.

### v2 API Chunked Upload Endpoints

| Step | Endpoint | XDK Works? |
|---|---|---|
| INIT | `POST /2/media/upload/initialize` | Yes (metadata only) |
| APPEND | `POST /2/media/upload/{mediaId}/append` | **No** (binary body) |
| FINALIZE | `POST /2/media/upload/{mediaId}/finalize` | Yes (no body) |

### OAuth1 Header Signing Helper

For the APPEND step, we sign only the HTTP method, URL, and OAuth parameters (not the JSON body -- per the OAuth1 spec, body params in `application/json` requests are not included in the signature base string).

```typescript
import crypto from "crypto";

interface OAuth1Credentials {
  apiKey: string;
  apiSecret: string;
  accessToken: string;
  accessTokenSecret: string;
}

function generateOAuth1Header(
  method: string,
  url: string,
  creds: OAuth1Credentials,
): string {
  const oauthParams: Record<string, string> = {
    oauth_consumer_key: creds.apiKey,
    oauth_nonce: crypto.randomBytes(16).toString("hex"),
    oauth_signature_method: "HMAC-SHA1",
    oauth_timestamp: Math.floor(Date.now() / 1000).toString(),
    oauth_token: creds.accessToken,
    oauth_version: "1.0",
  };

  // Signature base string: METHOD&URL&sorted_params (body NOT included for JSON requests)
  const paramString = Object.keys(oauthParams)
    .sort()
    .map((k) => `${encodeURIComponent(k)}=${encodeURIComponent(oauthParams[k])}`)
    .join("&");

  const baseString = [
    method.toUpperCase(),
    encodeURIComponent(url),
    encodeURIComponent(paramString),
  ].join("&");

  const signingKey = [
    encodeURIComponent(creds.apiSecret),
    encodeURIComponent(creds.accessTokenSecret),
  ].join("&");

  oauthParams.oauth_signature = crypto
    .createHmac("sha1", signingKey)
    .update(baseString)
    .digest("base64");

  const header = Object.keys(oauthParams)
    .sort()
    .map((k) => `${encodeURIComponent(k)}="${encodeURIComponent(oauthParams[k])}"`)
    .join(", ");

  return `OAuth ${header}`;
}
```

### Complete Upload Flow: Image with Post

```typescript
import fs from "fs";
import { Client, OAuth1 } from "@xdevplatform/xdk";

// --- Setup ---
const creds: OAuth1Credentials = {
  apiKey: process.env.X_API_KEY!,
  apiSecret: process.env.X_API_KEY_SECRET!,
  accessToken: process.env.X_ACCESS_TOKEN!,
  accessTokenSecret: process.env.X_ACCESS_TOKEN_SECRET!,
};

const oauth1 = new OAuth1({
  apiKey: creds.apiKey,
  apiSecret: creds.apiSecret,
  accessToken: creds.accessToken,
  accessTokenSecret: creds.accessTokenSecret,
});
const client = new Client({ oauth1 });

// --- Step 1: INIT via XDK (metadata only, no binary -- works fine) ---
const filePath = "image.jpg";
const fileBytes = fs.readFileSync(filePath);
const totalBytes = fileBytes.length;

const initResponse = await client.media.initializeUpload({
  totalBytes,
  mediaType: "image/jpeg",       // or "video/mp4", "image/png", etc.
  mediaCategory: "tweet_image",  // "tweet_video" for video, "tweet_gif" for GIF
});
const mediaId = initResponse.id;
console.log("INIT complete, mediaId:", mediaId);

// --- Step 2: APPEND via direct fetch (workaround for body-read bug) ---
const appendUrl = `https://api.x.com/2/media/upload/${mediaId}/append`;
const base64Media = fileBytes.toString("base64");

const appendResponse = await fetch(appendUrl, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: generateOAuth1Header("POST", appendUrl, creds),
  },
  body: JSON.stringify({
    media: base64Media,
    segment_index: 0,
  }),
});

if (!appendResponse.ok) {
  const errorText = await appendResponse.text();
  throw new Error(`APPEND failed (${appendResponse.status}): ${errorText}`);
}
console.log("APPEND complete");

// --- Step 3: FINALIZE via XDK (no body -- works fine) ---
const finalizeResponse = await client.media.finalizeUpload(mediaId);
console.log("FINALIZE complete:", finalizeResponse);

// --- Step 4: POST via XDK (small JSON body -- works fine) ---
const postResponse = await client.posts.create({
  text: "Hello with an image!",
  media: { media_ids: [mediaId] },
});
console.log("Post created:", postResponse.data?.id);
```

### Chunked Upload for Large Files (Video)

For files larger than a single chunk (e.g., video), split into multiple APPEND segments:

```typescript
const CHUNK_SIZE = 5 * 1024 * 1024; // 5 MB per chunk

// INIT
const videoBytes = fs.readFileSync("video.mp4");
const initRes = await client.media.initializeUpload({
  totalBytes: videoBytes.length,
  mediaType: "video/mp4",
  mediaCategory: "tweet_video",
});
const videoMediaId = initRes.id;

// APPEND each chunk
const totalChunks = Math.ceil(videoBytes.length / CHUNK_SIZE);
for (let i = 0; i < totalChunks; i++) {
  const chunk = videoBytes.subarray(i * CHUNK_SIZE, (i + 1) * CHUNK_SIZE);
  const appendUrl = `https://api.x.com/2/media/upload/${videoMediaId}/append`;

  const res = await fetch(appendUrl, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: generateOAuth1Header("POST", appendUrl, creds),
    },
    body: JSON.stringify({
      media: chunk.toString("base64"),
      segment_index: i,
    }),
  });

  if (!res.ok) {
    throw new Error(`APPEND chunk ${i} failed (${res.status}): ${await res.text()}`);
  }
  console.log(`APPEND chunk ${i + 1}/${totalChunks} complete`);
}

// FINALIZE
const finalRes = await client.media.finalizeUpload(videoMediaId);
console.log("FINALIZE complete:", finalRes);

// For video, poll processing status until complete
// (finalizeUpload may return a processing_info object with check_after_secs)
```

### Other Media Operations (Work Fine with OAuth1)

```typescript
// Get media by key
const media = await client.media.getByKey('3_1234567890', {
  mediaFields: ['url', 'preview_image_url', 'type'],
});

// Create metadata (alt text)
await client.media.createMetadata({
  media_id: mediaId,
  alt_text: { text: 'Description of the image' },
});

// Subtitles
await client.media.createSubtitles({
  media_id: mediaId,
  media_category: 'tweet_video',
  subtitle_info: {
    subtitles: [
      { language_code: 'en', display_name: 'English', media_id: subtitleMediaId },
    ],
  },
});

// Media analytics
const analytics = await client.media.getAnalytics({
  mediaKeys: ['key1', 'key2'],
});
```

## Lists

```typescript
// Create list
const list = await client.lists.create({
  name: 'My List',
  description: 'A curated list',
  private: false,
});

// Update list
await client.lists.update('listId', { name: 'Updated Name' });

// Delete list
await client.lists.delete('listId');

// Get list by ID
const listData = await client.lists.getById('listId');

// Get list members
const members = await client.lists.getMembers('listId', {
  userFields: ['username', 'public_metrics'],
});

// Get list tweets
const tweets = await client.lists.getTweets('listId', {
  tweetFields: ['created_at', 'public_metrics'],
});

// Add member
await client.lists.addMember('listId', { user_id: 'userId' });

// Remove member
await client.lists.removeMember('listId', 'userId');

// Get list followers
const listFollowers = await client.lists.getFollowers('listId');

// User's owned lists
const ownedLists = await client.users.getOwnedLists('userId');

// Follow a list
await client.users.followList('userId', { list_id: 'listId' });

// Unfollow a list
await client.users.unfollowList('userId', 'listId');

// Pin a list
await client.users.pinList('userId', { list_id: 'listId' });

// Unpin a list
await client.users.unpinList('userId', 'listId');
```

## Direct Messages

```typescript
// Create new conversation (group or 1-on-1)
const conversation = await client.directMessages.createConversation({
  conversation_type: 'Group',
  participant_ids: ['userId1', 'userId2'],
  message: { text: 'Hello everyone!' },
});

// Send message to existing conversation
await client.directMessages.createByConversationId('conversationId', {
  text: 'Follow-up message',
});

// Send message by participant ID (1-on-1)
await client.directMessages.createByParticipantId('userId', {
  text: 'Hey!',
});

// Get events by conversation
const events = await client.directMessages.getEventsByConversationId('conversationId', {
  maxResults: 50,
  dmEventFields: ['created_at', 'sender_id'],
});

// Get events by participant
const dmEvents = await client.directMessages.getEventsByParticipantId('userId');

// Get all DM events
const allEvents = await client.directMessages.getEvents();

// Get specific event
const event = await client.directMessages.getEventById('eventId');

// Delete DM event
await client.directMessages.deleteEvent('eventId');
```

## Spaces

```typescript
// Get Space by ID
const space = await client.spaces.getById('spaceId', {
  spaceFields: ['title', 'state', 'participant_count'],
});

// Get multiple Spaces
const spaces = await client.spaces.getByIds(['id1', 'id2']);

// Search Spaces
const results = await client.spaces.search('topic', {
  spaceFields: ['title', 'state'],
});

// Get posts from Space
const spacePosts = await client.spaces.getPosts('spaceId');

// Get Spaces by creator
const creatorSpaces = await client.spaces.getByCreatorIds(['userId']);

// Get ticket buyers
const buyers = await client.spaces.getBuyers('spaceId');
```

## Other Clients

```typescript
// Communities
const community = await client.communities.getById('communityId');
const communities = await client.communities.search('topic');

// Community Notes
await client.communityNotes.create({ text: 'Note text', tweet_id: 'postId' });
await client.communityNotes.evaluate({ note_id: 'noteId', rating: 'helpful' });

// Trends
const trends = await client.trends.getByWoeid(1); // 1 = worldwide
const personalTrends = await client.trends.getPersonalized();

// Usage
const usage = await client.usage.getTweetsUsage();

// Compliance
const job = await client.compliance.createJob({ type: 'tweets', name: 'my-job' });
const jobs = await client.compliance.getJobs({ type: 'tweets' });

// Webhooks
const webhook = await client.webhooks.create({ url: 'https://example.com/webhook' });
const webhooks = await client.webhooks.getAll();

// Connections
await client.connections.deleteAll();

// News
const news = await client.news.search('topic');
const article = await client.news.getById('articleId');
```

## Error Handling

```typescript
import { ApiError } from '@xdevplatform/xdk';

try {
  const post = await client.posts.getById('123');
} catch (error) {
  if (error instanceof ApiError) {
    console.error(`Status: ${error.statusCode}`);
    console.error(`Message: ${error.message}`);

    if (error.statusCode === 429) {
      // Rate limited - implement backoff
    }
    if (error.statusCode === 401) {
      // Authentication error - check credentials
    }
  }
}
```

## Request Options

Most methods accept an optional `requestOptions` parameter:

```typescript
const response = await client.posts.getById('123', {
  tweetFields: ['text'],
}, {
  // Request options
  raw: true, // Return raw HTTP response
});
```

## Full API Reference

For complete method signatures, parameters, and response types, see:
- [TypeScript client classes](https://github.com/xdevplatform/docs/tree/main/xdks/typescript/reference/classes) - All client class docs
- [TypeScript interfaces](https://github.com/xdevplatform/docs/tree/main/xdks/typescript/reference/interfaces) - All schema interfaces
- [Module index](https://github.com/xdevplatform/docs/blob/main/xdks/typescript/reference/modules.mdx) - Complete module listing
