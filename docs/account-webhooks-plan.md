# Account-Level Webhooks

## Problem

The existing webhook system is **board-scoped** — each webhook belongs to a single board and only fires for events on that board. There's no way to set up a centralized webhook that receives events from multiple (or all) boards in an account.

## Goal

Add an **account-level webhook** that:

1. Is configured centrally in Account Settings (admin-only)
2. Fires on the same event actions the existing system supports
3. Supports optional **board filtering** — select specific boards, or leave empty for all boards

## Domain Model

### New Tables

#### `account_webhooks`

| Column               | Type     | Notes                                      |
|----------------------|----------|--------------------------------------------|
| `id`                 | uuid     | Primary key (UUIDv7, base36)               |
| `account_id`         | uuid     | Belongs to account                         |
| `name`               | string   | Human-readable label                       |
| `url`                | text     | Payload destination URL (http/https)       |
| `signing_secret`     | string   | HMAC SHA256 signing key (`has_secure_token`)|
| `subscribed_actions` | text     | JSON-serialized array of action strings    |
| `active`             | boolean  | Default `true`                             |
| `created_at`         | datetime |                                            |
| `updated_at`         | datetime |                                            |

#### `account_webhook_board_filters`

| Column               | Type | Notes                        |
|----------------------|------|------------------------------|
| `id`                 | uuid | Primary key                  |
| `account_webhook_id` | uuid | Belongs to account webhook   |
| `board_id`           | uuid | Belongs to board             |

When the filter table is **empty** for a webhook, it fires for **all boards**. When populated, it only fires for events on the listed boards.

### New Model: `Account::Webhook`

```ruby
class Account::Webhook < ApplicationRecord
  include Account::Webhook::Triggerable

  belongs_to :account
  has_many :board_filters, class_name: "Account::Webhook::BoardFilter", dependent: :delete_all
  has_many :filtered_boards, through: :board_filters, source: :board
  has_many :deliveries, class_name: "Account::Webhook::Delivery", dependent: :delete_all
  has_one :delinquency_tracker, class_name: "Account::Webhook::DelinquencyTracker", dependent: :delete

  has_secure_token :signing_secret

  serialize :subscribed_actions, type: Array, coder: JSON

  scope :active, -> { where(active: true) }
end
```

### New Model: `Account::Webhook::BoardFilter`

```ruby
class Account::Webhook::BoardFilter < ApplicationRecord
  belongs_to :account_webhook, class_name: "Account::Webhook"
  belongs_to :board
end
```

## Delivery

### Triggering

Extend the existing `Event` lifecycle. When an event is created, in addition to dispatching board-level webhooks, also dispatch account-level webhooks.

```ruby
# app/models/event.rb
after_create_commit :dispatch_webhooks
after_create_commit :dispatch_account_webhooks

private
  def dispatch_webhooks
    Event::WebhookDispatchJob.perform_later(self)
  end

  def dispatch_account_webhooks
    Account::Webhook::DispatchJob.perform_later(self)
  end
```

### New Job: `Account::Webhook::DispatchJob`

Finds all active account webhooks that:
1. Match the event's action (via `subscribed_actions`)
2. Either have **no board filters** (= all boards) or have a filter matching the event's board

```ruby
class Account::Webhook::DispatchJob < ApplicationJob
  include ActiveJob::Continuable

  queue_as :webhooks
  discard_on ActiveJob::DeserializationError

  def perform(event)
    step :dispatch do |step|
      Account::Webhook.active.triggered_by(event).find_each(start: step.cursor) do |webhook|
        webhook.trigger(event)
        step.advance! from: webhook.id
      end
    end
  end
end
```

### Triggerable Concern

```ruby
module Account::Webhook::Triggerable
  extend ActiveSupport::Concern

  included do
    scope :triggered_by, ->(event) {
      triggered_by_action(event.action)
        .where(account: event.account)
        .filtering_board(event.board)
    }

    scope :triggered_by_action, ->(action) {
      where("subscribed_actions LIKE ?", "%\"#{action}\"%")
    }

    scope :filtering_board, ->(board) {
      left_joins(:board_filters)
        .where(account_webhook_board_filters: { board_id: [board.id, nil] })
        .distinct
    }
  end

  def trigger(event)
    deliveries.create!(event: event) unless account.cancelled?
  end
end
```

### Delivery Model: `Account::Webhook::Delivery`

Follows the same pattern as `Webhook::Delivery`:
- SSRF protection via `SsrfProtection.resolve_public_ip`
- HMAC SHA256 signature in `X-Webhook-Signature` header
- Timeout protection
- Reuses the same `webhooks/event.json.jbuilder` template for payload
- Tracks `pending → in_progress → completed/errored` state
- Has a `DelinquencyTracker` to auto-deactivate after repeated failures

## UI

### Account Settings Page

Add a new section to the account settings page (`app/views/account/settings/show.html.erb`), visible to admins only.

```erb
<div class="settings__panel panel shadow center">
  <%= render "account/settings/webhooks" %>
</div>
```

### Routes

```ruby
namespace :account do
  # existing...
  resources :webhooks do
    scope module: :webhooks do
      resource :activation, only: :create
    end
  end
end
```

### Controller: `Account::WebhooksController`

Standard CRUD, admin-only. Similar to `WebhooksController` but scoped to `Current.account` instead of a board.

### Views

| View                         | Purpose                                                       |
|------------------------------|---------------------------------------------------------------|
| `account/webhooks/index`     | List all account webhooks with status                         |
| `account/webhooks/new`       | Form: name, URL, board filter (multi-select), action checkboxes |
| `account/webhooks/show`      | Details, signing secret, deliveries log                       |
| `account/webhooks/edit`      | Edit name, board filter, actions (URL immutable after creation) |

The **board filter** is a multi-select of all boards in the account. Empty selection = all boards.

## Files to Create/Modify

### New Files

- `db/migrate/TIMESTAMP_create_account_webhooks.rb`
- `app/models/account/webhook.rb`
- `app/models/account/webhook/board_filter.rb`
- `app/models/account/webhook/triggerable.rb`
- `app/models/account/webhook/delivery.rb`
- `app/models/account/webhook/delinquency_tracker.rb`
- `app/jobs/account/webhook/dispatch_job.rb`
- `app/jobs/account/webhook/delivery_job.rb`
- `app/controllers/account/webhooks_controller.rb`
- `app/controllers/account/webhooks/activations_controller.rb`
- `app/views/account/webhooks/` (index, new, show, edit, partials)
- `app/views/account/settings/_webhooks.html.erb`
- `test/models/account/webhook_test.rb`
- `test/jobs/account/webhook/dispatch_job_test.rb`

### Modified Files

- `app/models/event.rb` — add `after_create_commit :dispatch_account_webhooks`
- `app/models/account.rb` — add `has_many :account_webhooks`
- `app/views/account/settings/show.html.erb` — render webhooks section
- `config/routes.rb` — add account webhook routes
- `config/recurring.yml` — add cleanup job for stale account webhook deliveries

## Reuse from Existing Webhook System

| Existing                          | Reuse                                             |
|-----------------------------------|---------------------------------------------------|
| `Webhook::PERMITTED_ACTIONS`      | Same action list                                  |
| `webhooks/event.json.jbuilder`    | Same JSON payload template                        |
| `webhooks/event.html.erb`         | Same HTML payload template                        |
| `SsrfProtection`                  | Same SSRF protection for delivery                 |
| `webhooks/_actions` form partial  | Extract shared partial for action checkboxes      |
| `WebhooksHelper`                  | Reuse `webhook_action_options` / `webhook_action_label` |
