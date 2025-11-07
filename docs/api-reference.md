## Telegram Component API Reference

This document describes the public surface area of the `aki\telegram\Telegram` component and explains how to configure, invoke, and extend it in a Yii 2 application.

The component wraps the [Telegram Bot API](https://core.telegram.org/bots/api) and adds production safeguards such as throttling awareness, message queueing, Redis-backed rate control, and optional SOCKS5 proxy support.

---

### Getting Started

- **Install via Composer**
  ```bash
  composer require alekseymikitin/yii2-bot-telegram:dev-master
  ```
- **Register the component**
  ```php
  // config/web.php
  return [
      'components' => [
          'telegram' => [
              'class' => 'aki\telegram\Telegram',
              'enabled' => true,
              'botToken' => getenv('TELEGRAM_BOT_TOKEN'),
              'reserveToken' => getenv('TELEGRAM_RESERVE_BOT_TOKEN'), // optional
              'botUsername' => 'MyAwesomeBot', // optional
              'apiURL' => 'https://api.telegram.org/bot',
              'redis' => 'redis',           // component ID or \yii\redis\Connection
              'proxy' => null,              // "<login>:<password>@<host>:<port>" for SOCKS5
          ],
          'redis' => [
              'class' => \yii\redis\Connection::class,
              'hostname' => '127.0.0.1',
              'port' => 6379,
              'database' => 0,
          ],
      ],
  ];
  ```
- **Basic usage**
  ```php
  Yii::$app->telegram->sendMessage([
      'chat_id' => $chatId,
      'text' => 'Hello from Yii 2!',
  ]);
  ```

---

### Component Configuration

| Property | Type | Required | Description |
| --- | --- | --- | --- |
| `botToken` | string | yes | Primary Bot API token obtained from `@BotFather`. |
| `reserveToken` | string|null | no | Secondary bot token used automatically while the primary token is throttled (HTTP 429). |
| `botUsername` | string|null | no | Human-readable bot username; not required for API calls. |
| `enabled` | bool | no | Shortcut switch. When `false`, outbound HTTP requests are skipped. Default: `false`. |
| `apiURL` | string | yes | Base API endpoint. Defaults to `https://api.telegram.org/bot`. |
| `hookContent` | array|null | no | Allows injecting decoded webhook payloads during testing. |
| `proxy` | string|null | no | SOCKS5 proxy string in the form `login:password@host:port`. |
| `redis` | string|\yii\redis\Connection | recommended | Handles cooldown tracking and message queueing. Provide a component ID or connection instance. |

> **Note:** A working Redis connection is mandatory when using throttling, message queueing, or dual-token failover (`sendMessage`, `flushQueue`, `viewQueue`).

---

### Rate Limiting & Queueing

- **Automatic 429 handling**  
  When Telegram responds with HTTP 429 and `Retry-After`, the component saves the cooldown in Redis (`telegram:retry_after_until`).

- **Primary vs. reserve token**  
  - Before the cooldown expires, the component switches to `reserveToken` (if provided) and prefixes sent messages with `RESERVE TOKEN`.
  - After the cooldown expires, queued messages are dispatched using the primary token.

- **Queue mechanics**
  - `sendMessage()` **never performs a direct API call.** Instead it enqueues the payload or, if throttled, sends it via the reserve token with an informational prefix.
  - Other methods enqueue themselves only when they hit rate limits inside `curl_call()`.

- **Operational tip**  
  Add a cron or background job to drain the queue:
  ```php
  // console/controllers/TelegramController.php
  public function actionFlushQueue(): void
  {
      Yii::$app->telegram->flushQueue();
  }
  ```

---

### Helper Methods

#### `flushQueue(): void`
- **Purpose:** Drains the Redis queue and dispatches queued messages based on the current throttling window.
- **Usage:**
  ```php
  Yii::$app->telegram->flushQueue();
  ```

#### `viewQueue(): array`
- **Purpose:** Inspect queued messages without dispatching them.
- **Returns:** Array of human-readable strings (`[method] at <timestamp>, params: {...}`).
- **Usage:**
  ```php
  foreach (Yii::$app->telegram->viewQueue() as $item) {
      Yii::info($item, 'telegram.queue');
  }
  ```

#### `hook(): void`
- **Purpose:** Convenience wrapper that loads webhook content into `hookContent`.
- **Usage:**
  ```php
  Yii::$app->telegram->hook();
  $update = Yii::$app->telegram->hookContent;
  ```

#### `getContentFromHook(): array|null`
- **Purpose:** Returns a decoded webhook payload (`php://input`), caching it in `hookContent`.
- **Usage:**
  ```php
  $update = Yii::$app->telegram->getContentFromHook();
  ```

#### `isCommand(array $message): bool`
- **Purpose:** Detects whether the first entity in a Telegram message is a bot command (`/start`, `/help`, ...).
- **Usage:**
  ```php
  if (Yii::$app->telegram->isCommand($update['message'])) {
      // handle command…
  }
  ```

#### `getFile(array $params)`
- **Parameters:** `file_id` (string, required)
- **Returns:** `stdClass` decoded response with file metadata.
- **Usage:**
  ```php
  $file = Yii::$app->telegram->getFile(['file_id' => $fileId]);
  $path = "https://api.telegram.org/file/bot{$token}/{$file->result->file_path}";
  ```

---

### Bot Information

#### `getMe()`
- **Purpose:** Retrieve basic information about the bot.
- **Returns:** Raw JSON string (decoded in callers as needed).
- **Usage:**
  ```php
  $botInfo = json_decode(Yii::$app->telegram->getMe(), true);
  ```

---

### Updates & Webhooks

#### `getUpdates(array $params = [])`
- **Parameters:** `offset`, `limit`, `timeout` (all optional)
- **Returns:** `stdClass` decoded response.
- **Usage:**
  ```php
  $updates = Yii::$app->telegram->getUpdates(['timeout' => 20]);
  ```

#### `setWebhook(array $params = [])`
- **Parameters:** `url` (string, required), plus optional fields supported by Telegram.
- **Usage:**
  ```php
  Yii::$app->telegram->setWebhook(['url' => 'https://example.com/telegram/webhook']);
  ```

---

### Messaging APIs

#### `sendMessage(array $params)`
- **Parameters:** `chat_id` (required), `text` (required), `parse_mode`, `reply_markup`, `disable_web_page_preview`, `reply_to_message_id`, etc.
- **Behavior:** Queues the message and dispatches it when permitted. Prefixes text with `RESERVE TOKEN` if fallback token is used immediately.
- **Usage:**
  ```php
  Yii::$app->telegram->sendMessage([
      'chat_id' => $chatId,
      'text' => '*Markdown* message',
      'parse_mode' => 'Markdown',
  ]);
  ```

#### `forwardMessage(array $params)`
- **Parameters:** `chat_id`, `from_chat_id`, `message_id`.
- **Returns:** `stdClass` response.
- **Usage:**
  ```php
  Yii::$app->telegram->forwardMessage([
      'chat_id' => $targetChat,
      'from_chat_id' => $sourceChat,
      'message_id' => $messageId,
  ]);
  ```

#### `sendChatAction(array $params)`
- **Parameters:** `chat_id`, `action` (`typing`, `upload_photo`, etc.).
- **Usage:**
  ```php
  Yii::$app->telegram->sendChatAction([
      'chat_id' => $chatId,
      'action' => 'typing',
  ]);
  ```

---

### Media & Attachment APIs

All media methods accept file paths or previously uploaded `file_id`s. Paths are converted via `curlFile()` before dispatch.

- **`sendPhoto(array $params)`**  
  Parameters: `chat_id`, `photo`, `caption`, `reply_markup`, etc.

- **`sendAudio(array $params)`**  
  Parameters: `chat_id`, `audio`, `caption`, `duration`, `performer`, `title`.

- **`sendDocument(array $params)`**  
  Parameters: `chat_id`, `document`, `caption`, …

- **`sendSticker(array $params)`**  
  Parameters: `chat_id`, `sticker`.

- **`sendVideo(array $params)`**  
  Parameters: `chat_id`, `video`, `caption`, `duration`, …

- **`sendLocation(array $params)`**  
  Parameters: `chat_id`, `latitude`, `longitude`, optional `live_period`, …

**Example**
```php
Yii::$app->telegram->sendPhoto([
    'chat_id' => $chatId,
    'photo' => Yii::getAlias('@app/runtime/test.jpg'),
    'caption' => 'Check out this photo',
]);
```

---

### User & Profile APIs

#### `getUserProfilePhotos(array $params)`
- **Parameters:** `user_id` (required), `offset`, `limit`.
- **Usage:**
  ```php
  $photos = Yii::$app->telegram->getUserProfilePhotos([
      'user_id' => $userId,
      'limit' => 5,
  ]);
  ```

---

### Chat Management APIs

Each method returns a decoded `stdClass` response unless stated otherwise.

- **`getChat(array $params)`** – Retrieve chat metadata.  
- **`getChatAdministrators(array $params)`** – List admins for a chat.  
- **`getChatMembersCount(array $params)`** – Count chat members.  
- **`getChatMember(array $params)`** – Inspect a specific member.  
- **`kickChatMember(array $params)`** – Ban a user (alias of `banChatMember` in newer Telegram API versions).  
- **`restrictChatMember(array $params)`** – Apply message restrictions.  
- **`promoteChatMember(array $params)`** – Promote a member to admin.  
- **`exportChatInviteLink(array $params)`** – Generate invite link.  
- **`deleteChatPhoto(array $params)`**, **`setChatTitle(array $params)`**, **`setChatDescription(array $params)`** – Update chat branding.  
- **`pinChatMessage(array $params)`**, **`unpinChatMessage(array $params)`** – Manage pinned messages.  
- **`leaveChat(array $params)`** – Make the bot leave a chat.  
- **`setChatStickerSet(array $params)`**, **`deleteChatStickerSet(array $params)`** – Attach/detach sticker set.

**Example**
```php
Yii::$app->telegram->restrictChatMember([
    'chat_id' => $chatId,
    'user_id' => $userId,
    'can_send_messages' => false,
    'until_date' => time() + 3600,
]);
```

---

### Callback & Inline APIs

- **`answerCallbackQuery(array $params)`** – Respond to inline keyboard button presses.  
  ```php
  Yii::$app->telegram->answerCallbackQuery([
      'callback_query_id' => $callbackId,
      'text' => 'Working on it…',
      'show_alert' => false,
  ]);
  ```

- **`editMessageText(array $params)`** – Update message text or disable previews.  
- **`editMessageCaption(array $params)`** – Update captions for sent media.  
- **`answerInlineQuery(array $params)`** – Provide inline results for `@YourBot <query>`. Requires pre-assembled result payloads.

---

### Games APIs

The component offers thin wrappers around Telegram’s game-related methods.

- **`sendGame(array $params)`** – Send a game.  
- **`Game(array $params)`** – Provide game metadata payloads.  
- **`Animation(array $params)`** – Attach animation metadata to a game.  
- **`CallbackGame(array $params)`** – Update scores for inline message games.  
- **`getGameHighScores(array $params)`** – Fetch scores for a user.  
- **`GameHighScore(array $params)`** – Compose individual score entries.

**Example**
```php
Yii::$app->telegram->sendGame([
    'chat_id' => $chatId,
    'game_short_name' => 'my_game',
    'disable_notification' => true,
    'reply_markup' => json_encode([
        'inline_keyboard' => [
            [['text' => 'Play', 'callback_game' => new stdClass()]],
        ],
    ]),
]);
```

---

### Best Practices

- **Guard outbound calls**: Always set `enabled => true` explicitly in production to avoid silent failures during testing.
- **Monitor queue length**: Use `viewQueue()` in diagnostics and set alerts if the queue grows unexpectedly (indicating sustained throttling or connectivity issues).
- **Validate inputs**: Telegram returns descriptive errors; capture them in logs to identify invalid parameters or permissions.
- **Use `flushQueue()` responsibly**: Invoke via cron or worker after a known cooldown window, or after switching tokens back from reserve to primary.
- **Proxy support**: When providing `proxy`, ensure the underlying cURL installation supports SOCKS5.

---

### Error Handling

- All API wrappers ultimately call `curl_call()`. Errors are logged to `@runtime/error_telegram.log`.
- When HTTP 429 is encountered:
  - `Retry-After` header is parsed.
  - Redis stores the cooldown (+10s cushion).
  - Subsequent `sendMessage` calls queue until cooldown ends.
- Non-recoverable cURL errors return `false`; always check responses before relying on them.

---

### Extending the Component

- **Custom Redis keys**: Override `$redisRetryKey` and `$redisQueueKey` in a subclass to isolate environments.
- **Alternate queue storage**: Replace `enqueueMessage()` and `flushQueue()` in a subclass if you prefer another queue backend.
- **New API wrappers**: Follow the existing pattern to add methods—construct the endpoint under `apiURL` + `token` + `/methodName` and pass parameters to `curl_call()`.

---

### Quick Reference

| Category | Methods |
| --- | --- |
| Bot info | `getMe` |
| Updates | `getUpdates`, `setWebhook` |
| Messaging & media | `sendMessage`, `forwardMessage`, `sendPhoto`, `sendAudio`, `sendDocument`, `sendSticker`, `sendVideo`, `sendLocation`, `sendChatAction` |
| User data | `getUserProfilePhotos`, `getFile`, `isCommand` |
| Chat management | `getChat`, `getChatAdministrators`, `getChatMembersCount`, `getChatMember`, `kickChatMember`, `restrictChatMember`, `promoteChatMember`, `exportChatInviteLink`, `deleteChatPhoto`, `setChatTitle`, `setChatDescription`, `pinChatMessage`, `unpinChatMessage`, `leaveChat`, `setChatStickerSet`, `deleteChatStickerSet` |
| Inline & callbacks | `answerCallbackQuery`, `answerInlineQuery`, `editMessageText`, `editMessageCaption` |
| Games | `sendGame`, `Game`, `Animation`, `CallbackGame`, `getGameHighScores`, `GameHighScore` |
| Queue management | `sendMessage`, `flushQueue`, `viewQueue` |
| Hooks | `hook`, `getContentFromHook` |

Refer to this reference whenever you integrate new Telegram Bot features or troubleshoot throttling behavior within your Yii 2 project.

