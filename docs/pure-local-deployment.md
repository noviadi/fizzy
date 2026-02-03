# Pure Local Deployment Guide

This guide covers deploying Fizzy on a VPS with Tailscale access, without any third-party services (no SMTP, no cloud storage, no external APIs).

## Architecture Overview

| Component | Solution |
|-----------|----------|
| Database | SQLite (local files) |
| File Storage | Local disk |
| Background Jobs | Solid Queue (database-backed) |
| Authentication | Manual user creation via console |
| Network Access | Tailscale (private mesh VPN) |
| Push Notifications | Disabled (optional) |

## Prerequisites

### System Requirements

- VPS with 1GB+ RAM (2GB recommended)
- Ubuntu 22.04+ / Debian 12+ (or similar)
- Root or sudo access

### 1. Install System Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install build essentials and libraries
sudo apt install -y \
  build-essential \
  git \
  curl \
  libssl-dev \
  libreadline-dev \
  zlib1g-dev \
  libsqlite3-dev \
  libffi-dev \
  libyaml-dev \
  imagemagick \
  libvips-dev
```

### 2. Install Ruby via mise

```bash
# Install mise (Ruby version manager)
curl https://mise.run | sh
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc
source ~/.bashrc

# Install Ruby 3.4.7
mise install ruby@3.4.7
mise use -g ruby@3.4.7

# Verify installation
ruby --version
# => ruby 3.4.7
```

### 3. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Note your Tailscale hostname
tailscale status
# Example: my-vps.tail12345.ts.net
```

## Application Setup

### 1. Clone Repository

```bash
cd ~
git clone <your-fizzy-repo-url> fizzy
cd fizzy
```

### 2. Install Ruby Gems

```bash
bundle config set --local deployment true
bundle install
```

### 3. Generate Master Key

```bash
# Generate a new master key
openssl rand -hex 16 > config/master.key
chmod 600 config/master.key

# Save this key somewhere safe - you'll need it for restarts
cat config/master.key
```

### 4. Configure for Tailscale Access

Add Tailscale hosts to the production configuration:

```bash
# Append to config/environments/production.rb (before final 'end')
cat >> config/environments/production.rb <<'RUBY'

  # Allow Tailscale hostnames
  config.hosts << /.*\.ts\.net/
  config.hosts << "localhost"
RUBY
```

### 5. Set Environment Variables

Create a production environment file:

```bash
cat > .env.production.local <<EOF
RAILS_ENV=production
DISABLE_SSL=true
BASE_URL=http://YOUR-VPS.tail12345.ts.net:3006
ACTIVE_STORAGE_SERVICE=local
FIZZY_DB_ADAPTER=sqlite
EOF
```

Replace `YOUR-VPS.tail12345.ts.net` with your actual Tailscale hostname.

### 6. Prepare Database

```bash
RAILS_ENV=production bin/rails db:prepare
```

This creates SQLite databases in the `storage/` directory:
- `storage/production.sqlite3` - Main database
- `storage/production_cable.sqlite3` - WebSocket/ActionCable
- `storage/production_cache.sqlite3` - Cache store
- `storage/production_queue.sqlite3` - Background jobs

## User Management

Since there's no SMTP configured, users must be created via Rails console.

### Create Admin User

```bash
RAILS_ENV=production bin/rails console
```

```ruby
# Create identity (global auth record)
identity = Identity.create!(email_address: "admin@local.dev")

# Create account with owner
account = Account.create_with_owner(
  account: {
    external_account_id: rand(10_000_000..99_999_999),
    name: "My Organization"
  },
  owner: {
    name: "Admin User",
    identity: identity
  }
)

# Generate magic link for login
link = identity.magic_links.create!(purpose: :sign_in)
host = "YOUR-VPS.tail12345.ts.net:3006"
puts "Login URL: http://#{host}/session/magic_links/#{link.token}/edit"
```

Copy the login URL and open it in your browser to sign in.

### Add Additional Users

```ruby
# In rails console
account = Account.first  # or find by name

# Create new user
new_identity = Identity.create!(email_address: "user@local.dev")
new_user = User.create!(
  name: "Team Member",
  identity: new_identity,
  account: account,
  role: :member,  # :owner, :admin, or :member
  verified_at: Time.current
)

# Generate their login link
link = new_identity.magic_links.create!(purpose: :sign_in)
puts "Login URL: http://#{host}/session/magic_links/#{link.token}/edit"
```

### Create Bot Users with API Access

```ruby
# Create bot identity
bot_identity = Identity.create!(email_address: "ci-bot@local.internal")

# Add to account
account = Account.first
bot_user = User.create!(
  name: "CI Bot",
  identity: bot_identity,
  account: account,
  role: :member,
  verified_at: Time.current
)

# Generate API token
token = bot_identity.access_tokens.create!(
  description: "CI/CD automation",
  permission: :write  # or :read for read-only
)

puts "API Token: #{token.token}"
puts "Account ID: #{account.external_account_id}"
```

Use the token in API requests:

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Accept: application/json" \
     "http://YOUR-VPS.tail12345.ts.net:3006/ACCOUNT_ID/boards"
```

## Running the Server

### Manual Start

```bash
cd ~/fizzy
RAILS_ENV=production bin/rails server -b 0.0.0.0 -p 3006
```

### Systemd Service (Recommended)

Create a systemd service for automatic startup:

```bash
sudo tee /etc/systemd/system/fizzy.service <<EOF
[Unit]
Description=Fizzy Project Management
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=$HOME/fizzy
Environment=RAILS_ENV=production
Environment=DISABLE_SSL=true
Environment=BASE_URL=http://YOUR-VPS.tail12345.ts.net:3006
Environment=ACTIVE_STORAGE_SERVICE=local
ExecStart=/bin/bash -lc 'bin/rails server -b 0.0.0.0 -p 3006'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable fizzy
sudo systemctl start fizzy

# Check status
sudo systemctl status fizzy

# View logs
sudo journalctl -u fizzy -f
```

## Verification

### Health Check

```bash
curl http://YOUR-VPS.tail12345.ts.net:3006/up
# Should return: 200 OK
```

### Test from Another Device

From any device connected to your Tailscale network:

1. Open browser to `http://YOUR-VPS.tail12345.ts.net:3006`
2. Use the magic link URL generated during user creation
3. You should see the Fizzy dashboard

### Test API Access

```bash
TOKEN="your-bot-token"
ACCOUNT="your-account-id"
HOST="YOUR-VPS.tail12345.ts.net:3006"

# List boards
curl -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/json" \
     "http://$HOST/$ACCOUNT/boards"

# Create a card
curl -X POST \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     -d '{"card": {"title": "Test card", "description": "<p>Created via API</p>"}}' \
     "http://$HOST/$ACCOUNT/boards/BOARD_ID/cards"
```

## Backup & Restore

### Backup

All data is stored in the `storage/` directory:

```bash
cd ~/fizzy

# Create backup
tar -czf fizzy-backup-$(date +%Y%m%d).tar.gz storage/

# Copy to safe location
scp fizzy-backup-*.tar.gz user@backup-server:/backups/
```

### Restore

```bash
cd ~/fizzy

# Stop the server
sudo systemctl stop fizzy

# Restore from backup
tar -xzf fizzy-backup-YYYYMMDD.tar.gz

# Restart
sudo systemctl start fizzy
```

## Database Locations

| Purpose | File Path |
|---------|-----------|
| Primary database | `storage/production.sqlite3` |
| WebSocket/Cable | `storage/production_cable.sqlite3` |
| Cache | `storage/production_cache.sqlite3` |
| Background jobs | `storage/production_queue.sqlite3` |
| File uploads | `storage/production/files/` |

## Troubleshooting

### "Blocked host" error

Ensure your Tailscale hostname is allowed in `config/environments/production.rb`:

```ruby
config.hosts << /.*\.ts\.net/
```

### Database errors

Reset and recreate the database:

```bash
RAILS_ENV=production bin/rails db:reset
```

### Permission errors on storage/

```bash
chmod -R 755 storage/
chown -R $USER:$USER storage/
```

### Check logs

```bash
# Application logs
tail -f log/production.log

# Systemd service logs
sudo journalctl -u fizzy -f
```

## Security Notes

1. **Tailscale provides encryption** - Traffic between devices is encrypted, so HTTPS is optional
2. **Keep master.key secret** - Never commit this file to version control
3. **API tokens are like passwords** - Store them securely, rotate periodically
4. **Magic links expire** - Generate new ones as needed via console

## What's Not Included

This deployment intentionally excludes:

- **SMTP/Email** - Users are created manually; no email notifications
- **Push notifications** - Requires VAPID keys and external push services
- **S3/Cloud storage** - Files stored locally on disk
- **Stripe/Payments** - SaaS billing features disabled
- **SSL certificates** - Tailscale handles encryption; add nginx/caddy if needed for public access
