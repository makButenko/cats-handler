# Cats Handler App

## Purpose

Cats handler consumes routed events from RabbitMQ and writes Telegram cat-related data into an inbox directory as files.

The handler is responsible for payload-specific processing. Unlike the router, it may inspect Telegram message fields, download Telegram files, group media albums, and create inbox folder structures.

## Architecture

```text
router-app
  -> RabbitMQ exchange inbox.handlers
  -> routing_key cats

cats-handler
  -> consumes RabbitMQ queue inbox.cats
  -> downloads Telegram files
  -> writes inbox files
```

## Responsibilities

Cats handler must:

- consume events from its RabbitMQ queue;
- parse the routed event envelope;
- inspect Telegram `update.message`;
- save raw event/update/message JSON;
- save text and captions as Markdown files;
- download supported Telegram files through the Bot API;
- append media album messages into folders by `media_group_id`;
- write a `manifest.json` index for every inbox item;
- acknowledge RabbitMQ only after files are safely written;
- handle duplicate deliveries idempotently;
- write structured logs;
- support graceful shutdown.

Cats handler must not:

- evaluate routing rules;
- read Redis;
- call the router directly;
- classify medical data;
- write into the smart knowledge base directly.

## RabbitMQ Input

The handler owns its queue and bindings.

Recommended topology:

```text
exchange: inbox.handlers
routing_key: cats
queue: inbox.cats

dead-letter exchange: inbox.handlers.dlx
dead-letter routing key: cats.dead
dead-letter queue: inbox.cats.dlq
```

The handler should use:

- durable queue;
- manual acknowledgements;
- bounded prefetch;
- dead-letter queue for permanently bad messages;
- idempotent processing.

The dead-letter queue prevents one poison message from blocking the main queue forever. Use it for messages that cannot be processed after inspection, such as invalid envelopes, malformed Telegram payloads, or files that intentionally exceed configured limits.

Processing rule:

```text
Receive RabbitMQ message -> write inbox files -> update manifest -> ack RabbitMQ
```

If processing fails before the current message is safely written, do not ack the message.

## Input Envelope

Expected routed event format:

```json
{
  "schema": "routed_event_v1",
  "event_id": "event:123456789",
  "rule_id": "cats-main-chat",
  "handler": "cats",
  "published_at": "2026-06-05T12:32:10Z",
  "routing": {
    "matched": true,
    "rule_id": "cats-main-chat",
    "rule_name": "Cats archive from family chat"
  },
  "update": {
    "update_id": 123456789,
    "message": {}
  }
}
```

The original Telegram update must be preserved in the `update` field.

## Telegram Message Handling

The handler should process these Telegram message fields:

```text
text
caption
photo
document
video
animation
audio
media_group_id
```

Temporarily ignore:

```text
voice
video_note
sticker
poll
location
contact
```

Ignored fields should not fail processing. Save the raw JSON and mark unsupported fields in the manifest when useful.

## Config Example

```yaml
app:
  name: cats-handler
  environment: production

rabbitmq:
  url_env: RABBITMQ_URL
  connection_name: cats-handler

  connection:
    heartbeat_seconds: 30
    connection_timeout_seconds: 10
    socket_timeout_seconds: 30
    retry:
      enabled: true
      max_attempts: 0
      initial_delay_ms: 1000
      max_delay_ms: 30000
      multiplier: 2.0

  consumer:
    queue: inbox.cats
    ack_mode: manual
    prefetch_count: 4
    consumer_tag: cats-handler
    requeue_on_error: false

  topology:
    declare_queue_on_startup: true
    passive_declare: false

  queue:
    name: inbox.cats
    durable: true
    exclusive: false
    auto_delete: false
    arguments:
      x-dead-letter-exchange: inbox.handlers.dlx
      x-dead-letter-routing-key: cats.dead
    bind:
      exchange: inbox.handlers
      routing_key: cats

  dead_letter_queue:
    name: inbox.cats.dlq
    durable: true
    exclusive: false
    auto_delete: false
    bind:
      exchange: inbox.handlers.dlx
      routing_key: cats.dead

telegram:
  bot_token_env: TELEGRAM_BOT_TOKEN
  api_base_url: https://api.telegram.org
  download_timeout_ms: 30000
  max_download_bytes: 52428800

inbox:
  root_dir: /srv/cats-inbox
  timezone: Europe/Samara
  manifest_name: manifest.json

  layout:
    date_format: "YYYY-MM-DD"
    time_format: "HH-mm-ss"
    item_folder_template: "{date}/{time}_msg-{message_id}"
    media_group_folder_template: "{date}/{time}_group-{media_group_id}"

  save:
    routed_event_json: true
    update_json: true
    message_json: true
    text_md: true
    caption_md: true
    files: true

media_groups:
  enabled: true
  mode: append_only

idempotency:
  strategy: manifest_index

errors:
  invalid_envelope: dead_letter
  unsupported_payload: save_raw_and_ack
  file_too_large: dead_letter
  telegram_download_error: retry
  filesystem_error: retry

runtime:
  shutdown_timeout_seconds: 60

logging:
  level: info
  format: json
```

## Inbox Layout

Every message or media group should become one inbox item folder.

Single text message:

```text
inbox/
  2026-06-05/
    14-30-01_msg-12344/
      manifest.json
      routed_event.json
      update.json
      message.json
      text.md
```

Single file message with caption:

```text
inbox/
  2026-06-05/
    14-32-10_msg-12345/
      manifest.json
      routed_event.json
      update.json
      message.json
      caption.md
      files/
        analysis.pdf
```

Media group:

```text
inbox/
  2026-06-05/
    14-35-20_group-137912381/
      manifest.json
      group.json
      captions.md
      messages/
        msg-12346.json
        msg-12347.json
      files/
        msg-12346_photo_1.jpg
        msg-12347_photo_1.jpg
```

## File Naming

Use Telegram `document.file_name` when available, after sanitizing it.

For Telegram photos, pick the largest available size from `message.photo`.

Recommended generated names:

```text
msg-{message_id}_photo_1.jpg
msg-{message_id}_document_{safe_original_name}
msg-{message_id}_video.mp4
msg-{message_id}_animation.mp4
msg-{message_id}_audio_{safe_original_name}
```

File names must be safe for the target filesystem:

- remove path separators;
- trim whitespace;
- replace control characters;
- avoid empty names;
- avoid overwriting existing files.

## Telegram File Download

Download flow:

1. Extract `file_id`.
2. Call `getFile`.
3. Read `file_path`.
4. Download from `/file/bot{token}/{file_path}`.
5. Write to a temporary file in the target item folder.
6. Rename temporary file to final file name after successful download.

The handler should store file metadata in `manifest.json`:

```json
{
  "files": [
    {
      "kind": "document",
      "telegram_file_id": "BAAC...",
      "telegram_file_unique_id": "AgAD...",
      "original_file_name": "analysis.pdf",
      "path": "files/analysis.pdf",
      "size": 123456,
      "mime_type": "application/pdf"
    }
  ]
}
```

## Media Group Handling

Telegram sends media groups as multiple messages with the same `media_group_id`.

The handler should not try to decide when a media group is complete. Telegram does not provide a reliable "last item" marker.

Instead, use append-only group folders:

```text
message.media_group_id exists -> open/create group folder -> append message and files -> update manifest -> ack
```

Default:

```yaml
media_groups:
  mode: append_only
```

The next processing layer decides when a group is old enough to process by looking at `manifest.json` `last_seen_at` or filesystem modification time.

RabbitMQ ack behavior for media groups:

- ack each RabbitMQ delivery after that message's JSON, files, captions, and manifest update are written;
- late media-group items append to the same group folder;
- on shutdown, finish the current write and leave unfinished deliveries unacked.

## Manifest

Every inbox item should have `manifest.json`.

The manifest is an index of folder contents. It is not a completion marker and must not contain a lifecycle status.

Example:

```json
{
  "schema": "cats_inbox_item_v1",
  "event_id": "event:123456789",
  "update_id": 123456789,
  "message_id": 12345,
  "media_group_id": null,
  "chat_id": 207568950,
  "from_id": 24353426,
  "message_date": "2026-06-05T14:32:10+04:00",
  "created_at": "2026-06-05T14:32:13+04:00",
  "item_type": "message",
  "has_text": false,
  "has_caption": true,
  "files": [
    {
      "kind": "document",
      "path": "files/analysis.pdf",
      "mime_type": "application/pdf",
      "size": 123456
    }
  ],
  "unsupported_fields": []
}
```

Write the manifest last for each message update.

## Idempotency

The handler must expect duplicate RabbitMQ deliveries.

Recommended idempotency keys:

```text
event_id
update.update_id
update.message.message_id
update.message.media_group_id
```

Before processing a single message:

1. Derive target item folder.
2. If `message_id` is already listed in `manifest.json`, treat the delivery as duplicate and ack.
3. If files exist but the message is not listed in the manifest, resume or rebuild that message's files and update the manifest.

Use temporary files for partial writes:

```text
filename.tmp
manifest.json.tmp
```

Then rename atomically to final names.

## Error Handling

### Invalid Envelope

Reject with `requeue=false` so RabbitMQ moves it to DLQ. Log the parse error and message metadata.

### Unsupported Message Payload

Save raw JSON if possible, write manifest with `unsupported_fields`, and ack.

Unsupported fields are not handler failures.

### Telegram Download Error

Do not ack until the file is downloaded or the message is intentionally skipped by the configured error policy.

If the error is permanent, reject with `requeue=false` and let RabbitMQ move the message to DLQ.

### Filesystem Error

Do not ack. Log the target path and error.

## Graceful Shutdown

On shutdown:

1. Stop consuming new RabbitMQ messages.
2. Finish current single-message writes.
3. Finish the current media-group append if one is in progress.
4. Ack only successfully written messages.
5. Close RabbitMQ channel and connection.
6. Exit before `shutdown_timeout_seconds`.

## Observability

Use structured JSON logs.

Each log event should include, when available:

```text
event_id
update_id
message_id
media_group_id
chat_id
from_id
item_folder
file_count
error
```

Recommended counters:

```text
cats_handler_messages_received_total
cats_handler_items_written_total
cats_handler_media_group_appends_total
cats_handler_files_downloaded_total
cats_handler_duplicates_total
cats_handler_unsupported_total
cats_handler_failures_total
cats_handler_skipped_total
cats_handler_dead_lettered_total
```

Optional HTTP endpoints:

```text
GET /health/live
GET /health/ready
GET /metrics
```

Readiness should check:

- RabbitMQ connectivity;
- Telegram token is configured;
- inbox root directory exists and is writable.

## Implementation Checklist

1. Load YAML config.
2. Validate config schema.
3. Connect to RabbitMQ.
4. Declare handler queue and binding if `declare_queue_on_startup` is true.
5. Start health endpoint if enabled.
6. Consume messages from `inbox.cats` with manual ack.
7. Parse routed event envelope.
8. Extract Telegram update/message.
9. Check whether `message_id` is already listed in the manifest.
10. If `media_group_id` exists, append to the group inbox item.
11. Otherwise, write a single-message inbox item.
12. Download Telegram files when present.
13. Write `manifest.json` last.
14. Ack RabbitMQ after successful write.
15. Nack with `requeue=true` for temporary failures.
16. Nack with `requeue=false` for permanent failures that should go to DLQ.
17. Handle shutdown gracefully.

## Explicitly Out of Scope

- routing rule evaluation;
- Redis reads;
- smart knowledge base updates;
- medical data classification;
- voice transcription;
- Kubernetes manifests;
- service discovery.
