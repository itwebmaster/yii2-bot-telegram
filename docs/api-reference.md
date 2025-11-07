# Telegram Component API Reference

This document describes the public API exposed by `aki\telegram\Telegram`. It expands on configuration, rate-limiting behaviour, and provides usage examples for every public method.

The component is designed for Yii 2 applications and wraps Telegram Bot API endpoints with additional resilience features such as Redis-backed queuing and reserve-token failover.

---

## Installation & Bootstrap

Install the extension via Composer:

```bash
composer require alekseymikitin/yii2-bot-telegram
```

Register the component in your Yii configuration:

```php
return [
    'components' => [
        'telegram' => [
            'class' => \aki\telegram\Telegram::class,
            'enabled' => true,                       // Set to false to disable outbound calls
            'botToken' => getenv('TELEGRAM_BOT_TOKEN'),
            'reserveToken' => getenv('TELEGRAM_RESERVE_TOKEN'), // Optional
            'apiURL' => 'https://api.telegram.org/bot',
            'redis' => 'redis',                      // Component ID or Connection instance
            'proxy' => 'user:pass@host:1080',        // Optional SOCKS5 proxy
        ],
    ],
];
```

Obtain your primary bot token from [BotFather](https://t.me/botfather). The reserve token is optional and used when the primary token is throttled (HTTP `429 Too Many Requests`).

---

## Configuration Reference

| Property      | Type                            | Required | Default | Description |
|---------------|---------------------------------|----------|---------|-------------|
| `botToken`    | `string`                        | Yes      | —       | Primary Telegram bot token. Used for all calls unless throttling forces a fallback. |
| `reserveToken`| `string`                        | No       | `null`  | Fallback bot token used after receiving `Retry-After`. When set, queued messages can be replayed via the reserve bot. |
| `botUsername` | `string`                        | No       | —       | Convenience field for your bot’s username. Not used internally. |
| `enabled`     | `bool`                          | No       | `false` | Toggles outbound HTTP requests. When `false`, all API methods return empty arrays without hitting Telegram. |
| `apiURL`      | `string`                        | Yes      | —       | Base URL for Telegram Bot API endpoints (typically `https://api.telegram.org/bot`). |
| `proxy`       | `string`                        | No       | `null`  | SOCKS5 proxy credentials in the format `login:password@host:port`. |
| `redis`       | `string|\yii\redis\Connection`  | No       | `'redis'` | Redis connection ID or instance, used for rate-limit coordination and the message queue. |
| `hookContent` | `array`                         | No       | `null`  | Cached payload of the last webhook. Populated by `getContentFromHook()`. |

The component keeps two Redis keys:

* `telegram:retry_after_until`: Timestamp (Unix) until which the primary token is throttled.
* `telegram:message_queue`: Redis list storing deferred call payloads.

---

## Rate Limiting, Queuing, and Reserve Token Workflow

Telegram imposes per-bot rate limits. To guard against `429 Too Many Requests` responses:

1. Every outgoing request passes through `curl_call()`. When a `429` is detected, the `Retry-After` header is parsed and the timestamp is stored in Redis (`telegram:retry_after_until`).
2. While throttled, the component automatically enqueues method payloads in `telegram:message_queue` when the target endpoint is a send-type method (e.g., `sendMessage`). Non-send calls triggered during throttling are attempted immediately using the reserve token if present.
3. Invoke `flushQueue()` periodically (e.g., via cron or a queue worker).  
   * If throttling has expired, queued messages are batched and pushed through the primary bot. Multiple `sendMessage` entries are combined into a single message prefixed with `RESERVE TOKEN` and separated by blank lines. Other payloads are replayed individually.  
   * If throttling is ongoing, each payload is replayed against the reserve bot with the `RESERVE TOKEN` prefix automatically added to message text.

**Recommended workflow**

```php
Yii::$app->telegram->sendMessage([...]); // Always queues first when main bot can send
Yii::$app->telegram->flushQueue();       // Trigger after each batch OR via scheduled job
```

Use `viewQueue()` to inspect pending items, especially when debugging.

---

## Return Types

Unless otherwise stated:

* Methods that perform API calls return the decoded JSON body (`stdClass` by default or `null` if decoding fails).
* `sendMessage()` returns `void` because it either enqueues or dispatches immediately through `curl_call()`.
* Utility methods such as `isCommand()` return plain PHP types.

---

## Public API Catalogue

Methods are grouped by domain. Each entry includes parameters and a ready-to-use snippet. All examples assume the component is registered as `Yii::$app->telegram`.

### Rate-Limit Utilities

#### `flushQueue(): void`
Flushes queued payloads from Redis. Respects the current throttle window and dispatches messages through the appropriate token.

```php
Yii::$app->telegram->flushQueue();
```

#### `viewQueue(): string[]`
Returns a human-readable summary of queued payloads.

```php
foreach (Yii::$app->telegram->viewQueue() as $line) {
    Yii::info($line, __METHOD__);
}
```

### Bot Information

#### `getMe()`
Returns basic information about the bot.

```php
$bot = Yii::$app->telegram->getMe();
if ($bot && $bot->ok) {
    Yii::info($bot->result->username, __METHOD__);
}
```

### Messaging

#### `sendMessage(array $option): void`
Queues or sends a text message. Required: `chat_id`, `text`. Optional: any Telegram `sendMessage` parameter (`reply_markup`, `parse_mode`, etc.). When the primary token is available it queues; call `flushQueue()` afterwards.

```php
Yii::$app->telegram->sendMessage([
    'chat_id' => $chatId,
    'text' => 'Hello from Yii!',
    'parse_mode' => 'MarkdownV2',
]);
Yii::$app->telegram->flushQueue();
```

#### `forwardMessage(array $option)`
Forwards a message. Required: `chat_id`, `from_chat_id`, `message_id`.

```php
$response = Yii::$app->telegram->forwardMessage([
    'chat_id' => $targetChat,
    'from_chat_id' => $sourceChat,
    'message_id' => $messageId,
]);
```

### Media Sending

All media methods accept `chat_id` plus media-specific parameters. Provide absolute file paths or file IDs. Files are converted to `CURLFile` instances automatically.

#### `sendPhoto(array $option)`

```php
$response = Yii::$app->telegram->sendPhoto([
    'chat_id' => $chatId,
    'photo' => '/path/to/image.jpg',
    'caption' => 'Check this out!',
]);
```

#### `sendAudio(array $option)`

```php
$response = Yii::$app->telegram->sendAudio([
    'chat_id' => $chatId,
    'audio' => '/path/to/audio.ogg',
    'caption' => 'Podcast episode',
    'duration' => 3600,
]);
```

#### `sendDocument(array $option)`

```php
$response = Yii::$app->telegram->sendDocument([
    'chat_id' => $chatId,
    'document' => '/path/to/manual.pdf',
    'caption' => 'Installation guide',
]);
```

#### `sendSticker(array $option)`

```php
$response = Yii::$app->telegram->sendSticker([
    'chat_id' => $chatId,
    'sticker' => '/path/to/sticker.webp',
]);
```

#### `sendVideo(array $option)`

```php
$response = Yii::$app->telegram->sendVideo([
    'chat_id' => $chatId,
    'video' => '/path/to/demo.mp4',
    'caption' => 'Demo clip',
    'duration' => 42,
]);
```

### Location & Actions

#### `sendLocation(array $option)`

```php
$response = Yii::$app->telegram->sendLocation([
    'chat_id' => $chatId,
    'latitude' => 37.7577,
    'longitude' => -122.5076,
    'disable_notification' => true,
]);
```

#### `sendChatAction(array $option)`

```php
Yii::$app->telegram->sendChatAction([
    'chat_id' => $chatId,
    'action' => 'typing',
]);
```

### Profile & Updates

#### `getUserProfilePhotos(array $option)`

```php
$photos = Yii::$app->telegram->getUserProfilePhotos([
    'user_id' => $userId,
    'offset' => 0,
    'limit' => 10,
]);
```

#### `getUpdates(array $option = [])`

```php
$updates = Yii::$app->telegram->getUpdates([
    'offset' => $lastUpdateId + 1,
    'timeout' => 30,
]);
```

#### `setWebhook(array $option = [])`

```php
$result = Yii::$app->telegram->setWebhook([
    'url' => 'https://example.com/bot/hook',
]);
```

### Chat Management

#### `getChat(array $option = [])`
Fetches chat metadata.

```php
$chat = Yii::$app->telegram->getChat(['chat_id' => $chatId]);
```

#### `getChatAdministrators(array $option = [])`
Lists current administrators for a supergroup or channel.

```php
$admins = Yii::$app->telegram->getChatAdministrators(['chat_id' => $chatId]);
```

#### `getChatMembersCount(array $option = [])`
Returns the number of members in a chat.

```php
$count = Yii::$app->telegram->getChatMembersCount(['chat_id' => $chatId]);
```

#### `getChatMember(array $option = [])`
Retrieves information about a specific member in a chat.

```php
$member = Yii::$app->telegram->getChatMember([
    'chat_id' => $chatId,
    'user_id' => $userId,
]);
```

#### `kickChatMember(array $option = [])`
Temporarily or permanently bans a user.

```php
Yii::$app->telegram->kickChatMember([
    'chat_id' => $chatId,
    'user_id' => $userId,
    'until_date' => time() + 7 * 86400,
]);
```

#### `restrictChatMember(array $option = [])`
Applies granular messaging restrictions to a user.

```php
Yii::$app->telegram->restrictChatMember([
    'chat_id' => $chatId,
    'user_id' => $userId,
    'can_send_messages' => false,
    'until_date' => time() + 3600,
]);
```

#### `promoteChatMember(array $option = [])`
Grants admin privileges to a user.

```php
Yii::$app->telegram->promoteChatMember([
    'chat_id' => $chatId,
    'user_id' => $userId,
    'can_pin_messages' => true,
    'can_manage_chat' => true,
]);
```

#### `exportChatInviteLink(array $option = [])`
Generates a new invite link for a chat.

```php
$link = Yii::$app->telegram->exportChatInviteLink(['chat_id' => $chatId]);
```

#### `deleteChatPhoto(array $option = [])`
Removes the chat’s photo.

```php
Yii::$app->telegram->deleteChatPhoto(['chat_id' => $chatId]);
```

#### `setChatTitle(array $option = [])`
Updates the chat title.

```php
Yii::$app->telegram->setChatTitle([
    'chat_id' => $chatId,
    'title' => 'New Title',
]);
```

#### `setChatDescription(array $option = [])`
Updates the chat description.

```php
Yii::$app->telegram->setChatDescription([
    'chat_id' => $chatId,
    'description' => 'Community guidelines',
]);
```

#### `pinChatMessage(array $option = [])`
Pins a message to the top of the chat.

```php
Yii::$app->telegram->pinChatMessage([
    'chat_id' => $chatId,
    'message_id' => $messageId,
]);
```

#### `unpinChatMessage(array $option = [])`
Unpins the current pinned message.

```php
Yii::$app->telegram->unpinChatMessage(['chat_id' => $chatId]);
```

#### `leaveChat(array $option = [])`
Makes the bot leave the chat.

```php
Yii::$app->telegram->leaveChat(['chat_id' => $chatId]);
```

#### `setChatStickerSet(array $option = [])`
Assigns a sticker set to the supergroup.

```php
Yii::$app->telegram->setChatStickerSet([
    'chat_id' => $chatId,
    'sticker_set_name' => 'FunPack',
]);
```

#### `deleteChatStickerSet(array $option = [])`
Removes the current sticker set.

```php
Yii::$app->telegram->deleteChatStickerSet(['chat_id' => $chatId]);
```

### Callback & Inline Interactions

#### `answerCallbackQuery(array $option = [])`

```php
Yii::$app->telegram->answerCallbackQuery([
    'callback_query_id' => $callbackId,
    'text' => 'Processing...',
    'show_alert' => false,
]);
```

#### `editMessageText(array $option = [])`

```php
Yii::$app->telegram->editMessageText([
    'chat_id' => $chatId,
    'message_id' => $messageId,
    'text' => 'Updated text',
    'parse_mode' => 'HTML',
]);
```

#### `editMessageCaption(array $option = [])`

```php
Yii::$app->telegram->editMessageCaption([
    'chat_id' => $chatId,
    'message_id' => $messageId,
    'caption' => 'Updated caption',
]);
```

#### `answerInlineQuery(array $option = [])`

```php
Yii::$app->telegram->answerInlineQuery([
    'inline_query_id' => $inlineQueryId,
    'results' => json_encode($results),
    'cache_time' => 300,
]);
```

### Games API

#### `sendGame(array $option = [])`
Sends a game to the specified chat.

```php
Yii::$app->telegram->sendGame([
    'chat_id' => $chatId,
    'game_short_name' => 'lumberjack',
    'disable_notification' => false,
]);
```

#### `Game(array $option = [])`
Uploads or updates a game configuration payload.

```php
Yii::$app->telegram->Game([
    'title' => 'My Game',
    'description' => 'Collect coins!',
    'photo' => $photoArray,
]);
```

#### `Animation(array $option = [])`
Provides animation metadata for a game.

```php
Yii::$app->telegram->Animation([
    'file_id' => $animationFileId,
    'thumb' => $thumbPhoto,
    'file_name' => 'intro.mp4',
]);
```

#### `CallbackGame(array $option = [])`
Sets a user’s score in a game.

```php
Yii::$app->telegram->CallbackGame([
    'user_id' => $userId,
    'score' => 123,
    'force' => true,
    'chat_id' => $chatId,
    'message_id' => $messageId,
]);
```

#### `getGameHighScores(array $option = [])`
Retrieves game high scores near a given player.

```php
$scores = Yii::$app->telegram->getGameHighScores([
    'user_id' => $userId,
    'chat_id' => $chatId,
    'message_id' => $messageId,
]);
```

#### `GameHighScore(array $option = [])`
Creates a high-score payload (utility wrapper).

```php
$entry = Yii::$app->telegram->GameHighScore([
    'position' => 1,
    'user' => $userObject,
    'score' => 999,
]);
```

### File Handling

#### `getFile(array $option)`

```php
$file = Yii::$app->telegram->getFile([
    'file_id' => $fileId,
]);
```

Use the returned `file_path` with `https://api.telegram.org/file/bot<token>/<file_path>` to download media.

### Webhook Processing Helpers

#### `hook(): void`
Convenience wrapper that reads and caches the incoming webhook payload by calling `getContentFromHook()`.

```php
public function actionHook(): void
{
    Yii::$app->telegram->hook();
    $payload = Yii::$app->telegram->getContentFromHook();
    // Handle $payload...
}
```

#### `getContentFromHook(): array|null`
Reads and caches the webhook payload (`php://input`). Subsequent calls return the cached array.

```php
$payload = Yii::$app->telegram->getContentFromHook();
if (!empty($payload['message'])) {
    // Handle incoming message
}
```

### Utilities

#### `isCommand(array $message): bool`
Returns `true` when the message contains an entity of type `bot_command`.

```php
if (Yii::$app->telegram->isCommand($payload['message'] ?? [])) {
    // Dispatch to command router
}
```

---

## Implementation Notes

* **Enabling & disabling**: Set `enabled` to `false` in staging environments to disable external calls while preserving method contracts.
* **Error logging**: Transport errors (e.g., network failures) are appended to `@runtime/error_telegram.log`.
* **Proxy support**: Set the `proxy` property to route all requests through a SOCKS5 proxy—useful for environments where Telegram is blocked.
* **Binary attachments**: When you pass local file paths, the component automatically converts them into `CURLFile` instances (`CURLFile` class is required in PHP ≥ 5.5).

---

## Debugging Checklist

- Call `viewQueue()` to ensure your queue is draining as expected.
- Inspect Redis for `telegram:retry_after_until` to understand throttle windows.
- Confirm that `enabled` is `true` before troubleshooting missing messages.
- When using a reserve token, monitor both bots’ chats to verify where messages land.

For further details on request parameters and responses, consult the official [Telegram Bot API documentation](https://core.telegram.org/bots/api).

