# NanoClaw Migration Guide

Generated: 2026-04-24
Base: a81e1651b5e48c9194162ffa2c50a22283d5ecd3
HEAD at generation: fd4ceb0552ae5d39a56ae8b5c22cef59bc6b9362
Upstream: a4346f566c87a25418aa5e783fc2a54089e11e6a
Migrated: 2026-04-24
HEAD after migration: 1bf40326ad11ba7b279e620348f48417fa21ee8c

## Summary

This fork's only customization is the **Telegram channel**. The channel came from
the external `nanoclaw-telegram` fork (`https://github.com/qwibitai/nanoclaw-telegram.git`)
and was then extended with reply context persistence, file downloads, thread/topic ID
support, @mention translation, and bot commands.

In v2, the Telegram adapter is a complete architectural rewrite — it uses the
`@chat-adapter/telegram` Chat SDK adapter + `createChatSdkBridge` abstraction instead of
grammy directly. The upgrade path is: apply `/add-telegram` (which copies from the
`channels` branch), then port the custom features into the new architecture.

## Applied Skills

None from upstream skill branches. All customizations are Telegram-only.

The Telegram channel is **not** from an upstream skill branch — it came from the external
fork `https://github.com/qwibitai/nanoclaw-telegram.git`. In v2, Telegram is installed
via the `/add-telegram` skill, which copies files from the `channels` branch and installs
`@chat-adapter/telegram`. The grammy dependency (`grammy@^1.39.3`) from this fork should
be removed after the v2 Telegram is installed.

## Customizations

### 1. Install Telegram channel (v2 approach)

**Intent:** Add Telegram bot support. In v2 this is done via the channels branch, not the
external telegram fork.

**Files:** `src/channels/telegram.ts`, `src/channels/telegram-pairing.ts`,
`src/channels/telegram-markdown-sanitize.ts`, and their `.test.ts` siblings,
`setup/pair-telegram.ts`, `src/channels/index.ts`, `setup/index.ts`, `package.json`

**How to apply:**

Run the `/add-telegram` skill which will:
1. Fetch the `channels` branch
2. Copy the v2 adapter and helpers into `src/channels/`
3. Add the import to `src/channels/index.ts`
4. Register the setup step in `setup/index.ts`
5. Install `@chat-adapter/telegram` via pnpm

After the skill runs, remove the legacy grammy dependency:
```bash
pnpm remove grammy
```

The v2 `src/channels/telegram.ts` will be ~229 lines using `registerChannelAdapter`
and `createChatSdkBridge`. This replaces the old grammy-based `TelegramChannel` class.

---

### 2. Extended reply context — include message ID

**Intent:** When an agent receives a reply to a message, it should know not just the text
and sender of the original message, but also its message ID (for referencing and DB
storage). The user had three extra fields: `reply_to_message_id`, `reply_to_message_content`,
`reply_to_sender_name`.

**Files:** `src/channels/telegram.ts`, `src/channels/chat-sdk-bridge.ts` (or just telegram.ts if ReplyContext is extended locally)

**Background:** In v2, the `extractReplyContext` function in the installed telegram.ts returns:
```typescript
{ text: string; sender: string }
```
The v2 `ReplyContext` interface (in `chat-sdk-bridge.ts`) only has `{ text, sender }`.

**How to apply:**

In `src/channels/telegram.ts`, update the `extractReplyContext` function to also return `id`:

```typescript
// eslint-disable-next-line @typescript-eslint/no-explicit-any
function extractReplyContext(raw: Record<string, any>): ReplyContext | null {
  if (!raw.reply_to_message) return null;
  const reply = raw.reply_to_message;
  return {
    id: String(reply.message_id ?? ''),          // ADD THIS LINE
    text: reply.text || reply.caption || '',
    sender: reply.from?.first_name || reply.from?.username || 'Unknown',
  };
}
```

Then extend the `ReplyContext` interface in `src/channels/chat-sdk-bridge.ts`:
```typescript
export interface ReplyContext {
  id?: string;      // ADD THIS
  text: string;
  sender: string;
}
```

**Note:** Also verify that wherever the bridge stores reply context in the DB, the `id`
field is persisted. Search for `reply_to` in `src/db/` and `src/ipc.ts` to find where
reply context flows downstream.

---

### 3. File downloads — photos, videos, audio, documents

**Intent:** When a user sends a media attachment in Telegram, the bot downloads it to the
group's `attachments/` directory on disk and delivers the container-relative path to the
agent (e.g. `/workspace/group/attachments/photo_123.jpg`). Falls back to a placeholder
string if the download fails.

**Files:** `src/channels/telegram.ts`

**Background:** The v2 telegram adapter does not include file download logic. It only
handles text messages. This feature needs to be added back.

**How to apply:**

Add the following private helper method and media handlers to the v2 telegram.ts. This
requires importing `fs`, `path`, `https`, and `resolveGroupFolderPath`.

Add imports at the top of `src/channels/telegram.ts`:
```typescript
import fs from 'node:fs';
import https from 'node:https';
import path from 'node:path';
import { resolveGroupFolderPath } from '../group-folder.js';
```

Add the `downloadFile` helper (can be a module-level function, not a class method since v2
uses a functional style):
```typescript
async function downloadFile(
  botToken: string,
  fileId: string,
  groupFolder: string,
  filename: string,
): Promise<string | null> {
  try {
    const metaRes = await fetch(`https://api.telegram.org/bot${botToken}/getFile?file_id=${fileId}`);
    const meta = (await metaRes.json()) as { ok: boolean; result?: { file_path?: string } };
    if (!meta.ok || !meta.result?.file_path) return null;

    const groupDir = resolveGroupFolderPath(groupFolder);
    const attachDir = path.join(groupDir, 'attachments');
    fs.mkdirSync(attachDir, { recursive: true });

    const tgExt = path.extname(meta.result.file_path);
    const localExt = path.extname(filename);
    const safeName = filename.replace(/[^a-zA-Z0-9._-]/g, '_');
    const finalName = localExt ? safeName : `${safeName}${tgExt}`;
    const destPath = path.join(attachDir, finalName);

    const fileUrl = `https://api.telegram.org/file/bot${botToken}/${meta.result.file_path}`;
    const resp = await fetch(fileUrl);
    if (!resp.ok) return null;

    fs.writeFileSync(destPath, Buffer.from(await resp.arrayBuffer()));
    return `/workspace/group/attachments/${finalName}`;
  } catch {
    return null;
  }
}
```

The v2 adapter uses a lower-level event model. You will need to look at how the v2
`createTelegramAdapter` exposes raw message events for media types and hook into them.
The approach depends on what `@chat-adapter/telegram` exposes. Check its API or look at
how `message.raw` is populated in the bridge — that's the grammy context object.

**Alternative approach:** If `@chat-adapter/telegram` doesn't expose grammy event hooks
for media, consider filing an issue or handling it via the raw grammy bot instance
that the adapter wraps.

---

### 4. Thread / topic ID support

**Intent:** When a message comes from a Telegram supergroup topic (forum), the
`message_thread_id` is captured and forwarded as `thread_id` so the agent can reply back
to the same topic thread.

**Files:** `src/channels/telegram.ts`

**Background:** The v2 telegram.ts calls `createChatSdkBridge` with `supportsThreads: false`.
The user's grammy-based implementation extracted `ctx.message.message_thread_id` and
passed it as `thread_id`.

**How to apply:**

Change `supportsThreads: false` to `supportsThreads: true` in the bridge config inside
`src/channels/telegram.ts`:

```typescript
const bridge = createChatSdkBridge({
  adapter: telegramAdapter,
  concurrency: 'concurrent',
  extractReplyContext,
  supportsThreads: true,   // was false
  transformOutboundText: sanitizeTelegramLegacyMarkdown,
});
```

Then verify that `@chat-adapter/telegram` actually extracts `message_thread_id` and maps
it to the thread ID when `supportsThreads` is true. If it doesn't, you may need to add
a custom extraction step. Check the adapter's source or changelog for thread support notes.

---

### 5. @mention translation — bridge Telegram @bot_username to TRIGGER_PATTERN

**Intent:** Telegram @mentions (e.g. `@andy_ai_bot`) don't match NanoClaw's
`TRIGGER_PATTERN` (e.g. `^@Andy\b`). When the bot is @mentioned, the message is
rewritten to prepend the trigger (`@Andy`) before being delivered to the agent.

**Files:** `src/channels/telegram.ts`

**Background:** The v2 adapter does not include this logic. In v2, trigger detection
may be handled differently (e.g. via the Chat SDK bridge's `isMention` flag). Check
whether the v2 bridge already handles `@mention` → trigger detection natively before
adding this back.

**How to apply (if not already handled by v2):**

In `src/channels/telegram.ts`, after the bridge processes the message but before it
reaches the router, add a transform that detects Telegram mention entities and prepends
the NanoClaw trigger when the bot is @mentioned. This would likely go in a wrapper
around the bridge's `onInbound` — similar to the pairing interceptor pattern used in v2.

The core logic from the v1 implementation:
```typescript
import { ASSISTANT_NAME, TRIGGER_PATTERN } from '../config.js';

// Inside the inbound message handler:
const botUsername = /* fetch from Telegram getMe */;
const entities = raw.entities || [];
const isBotMentioned = entities.some((entity: any) => {
  if (entity.type === 'mention') {
    const mentionText = text.substring(entity.offset, entity.offset + entity.length).toLowerCase();
    return mentionText === `@${botUsername.toLowerCase()}`;
  }
  return false;
});
if (isBotMentioned && !TRIGGER_PATTERN.test(text)) {
  text = `@${ASSISTANT_NAME} ${text}`;
}
```

**Note:** The bot username can be fetched once via `https://api.telegram.org/bot${token}/getMe`
and cached (the v2 telegram.ts already has `fetchBotUsername` for the pairing interceptor).

---

### 6. Bot commands: /chatid and /ping

**Intent:** The Telegram bot responds to `/chatid` (returns the chat's registration ID
in `tg:NUMERIC_ID` format) and `/ping` (confirms the bot is online). These help users
discover their chat ID for group registration.

**Files:** `src/channels/telegram.ts`

**Background:** The v2 Chat SDK adapter may not support bot commands natively. This
feature may require accessing the underlying grammy bot instance.

**How to apply:**

Check if `@chat-adapter/telegram` exposes a way to register grammy commands. If it
exposes the underlying grammy `Bot` instance (check the adapter's API), use:

```typescript
// Register before starting the adapter
telegramAdapter.bot.command('chatid', (ctx) => {
  const chatId = ctx.chat.id;
  const chatType = ctx.chat.type;
  const chatName =
    chatType === 'private'
      ? ctx.from?.first_name || 'Private'
      : (ctx.chat as any).title || 'Unknown';
  ctx.reply(
    `Chat ID: \`tg:${chatId}\`\nName: ${chatName}\nType: ${chatType}`,
    { parse_mode: 'Markdown' },
  );
});
telegramAdapter.bot.command('ping', (ctx) => {
  ctx.reply(`${ASSISTANT_NAME} is online.`);
});
```

If `@chat-adapter/telegram` doesn't expose the grammy bot instance, this feature may
need to wait for an upstream enhancement or a PR to the chat adapter package.

---

### 7. types.ts — extended NewMessage fields

**Intent:** The `NewMessage` type (in `src/types.ts`) was extended with four optional
fields to carry Telegram-specific data downstream.

**Files:** `src/types.ts`

**How to apply:**

In `src/types.ts`, find the `NewMessage` interface (or type alias) and add:

```typescript
thread_id?: string;              // Telegram topic/forum message_thread_id
reply_to_message_id?: string;   // ID of the message being replied to
reply_to_message_content?: string; // Text/caption of the replied-to message
reply_to_sender_name?: string;  // Display name of the sender of the replied-to message
```

**Note:** In v2, the type system is significantly refactored. Verify that `NewMessage`
still exists with this name. If the v2 uses a different inbound message type (e.g.
from `chat-sdk-bridge.ts`'s `InboundMessage`), find the right interface to extend.
Search for `thread_id` in the v2 codebase to see where threads flow.

---

### 8. Suppress upstream GitHub Actions auto-sync workflows

**Intent:** The upstream ships `.github/workflows/bump-version.yml` and
`.github/workflows/update-tokens.yml`. These were intentionally deleted from this fork
(commit `e9426da`: "chore: remove auto-sync GitHub Actions") because they are only
intended for the upstream's own repo (they have a `github.repository == 'qwibitai/nanoclaw'`
guard, but they're noisy on a personal fork).

**Files:** `.github/workflows/bump-version.yml`, `.github/workflows/update-tokens.yml`

**How to apply:**

After the upgrade, check if these files reappear. If so, delete them:
```bash
rm -f .github/workflows/bump-version.yml .github/workflows/update-tokens.yml
```

The upstream `ci.yml` is fine to keep — it's the actual test/lint CI. Only the
bump-version and update-tokens workflows need to be removed.
