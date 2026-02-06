# Create Bot Accounts

This guide creates all bot accounts for the agent team. Run all scripts in Rails console:

```bash
RAILS_ENV=production bin/rails console
```

## Bot Roster

| Name | Email | Role | Description |
|------|-------|------|-------------|
| Pak Lurah | pak-lurah@local.internal | Orchestrator | Main orchestrator that assigns tasks to other agents |
| Yanto | yanto@local.internal | Farmer | Deals with farming tasks |
| Markonah | markonah@local.internal | Family Assistant | Personal family assistant |
| Cipto | cipto@local.internal | Engineer | Hacker/engineer that does development |
| Sri | sri@local.internal | Finance Minister | Finance and trade minister |
| Udin | udin@local.internal | Researcher | Research and knowledge gathering |

## Create All Bots

```ruby
account = Account.first

bots = [
  { name: "Pak Lurah",  email: "pak-lurah@local.internal",  avatar: "pak-lurah-avatar.png",  desc: "Orchestrator bot" },
  { name: "Yanto",      email: "yanto@local.internal",      avatar: "yanto-avatar.png",      desc: "Farming bot" },
  { name: "Markonah",   email: "markonah@local.internal",    avatar: "markonah-avatar.png",   desc: "Family assistant bot" },
  { name: "Cipto",      email: "cipto@local.internal",       avatar: "cipto-avatar.png",      desc: "Engineer/hacker bot" },
  { name: "Sri",        email: "sri@local.internal",         avatar: "sri-avatar.png",        desc: "Finance and trade bot" },
  { name: "Udin",       email: "udin@local.internal",       avatar: "udin-avatar.png",       desc: "Researcher bot" },
]

bots.each do |bot|
  identity = Identity.find_or_create_by!(email_address: bot[:email])
  identity.join(account, name: bot[:name], role: :member, verified_at: Time.current)

  avatar_path = Rails.root.join("tmp", bot[:avatar])
  if File.exist?(avatar_path) && !identity.avatar.attached?
    identity.avatar.attach(
      io: File.open(avatar_path),
      filename: bot[:avatar],
      content_type: "image/png"
    )
  end

  token = identity.access_tokens.first || identity.access_tokens.create!(
    description: bot[:desc],
    permission: :write
  )

  puts "#{bot[:name]}: token=#{token.token}"
end

puts "\nAccount ID: #{account.external_account_id}"
```

## Generate Login Links

To log in as any bot interactively:

```ruby
email = "cipto@local.internal"  # change to desired bot
identity = Identity.find_by(email_address: email)
link = identity.magic_links.create!(purpose: :sign_in)
host = "YOUR-VPS.tail12345.ts.net:3006"
puts "Login URL: http://#{host}/session/magic_links/#{link.token}/edit"
```

## Avatar Files

Avatars are stored in `tmp/` and must be present before running the creation script:

| Bot | Avatar File |
|-----|-------------|
| Pak Lurah | `tmp/pak-lurah-avatar.png` |
| Yanto | `tmp/yanto-avatar.png` |
| Markonah | `tmp/markonah-avatar.png` |
| Cipto | `tmp/cipto-avatar.png` |
| Sri | `tmp/sri-avatar.png` |
| Udin | `tmp/udin-avatar.png` |
