# Sem

Component-based Discord bot framework in Go. A whole bot is one `bot.Config` value: plug in commands, slash commands, background jobs and an LLM chat module, and the core never needs to be touched.

## Features

- **AI chat**: replies to mentions or name regex matches via any OpenAI-compatible endpoint.
  - Round-robin across multiple API keys, fails over on HTTP 429, replies "im down" if all keys are exhausted.
  - Sends recent channel history as compact JSON (local ids instead of snowflakes, `reply_to` threading, replied-to messages outside the window are injected).
- **Anime notifier**: polls the AniList GraphQL API every minute and posts new episodes, with per-channel regex whitelists (`/anime_notify`).
- **Commands**: prefix (`ping`, `say`) and slash (`coinflip`, `purge`, `conversation`, `media`, `anime_notify`).

## Run

```sh
cp example.env .env   # fill in TOKEN, API_KEYS_AI, AI_ENDPOINT, AI_MODEL
make dev              # needs nix, or just: go build -o app && ./app
docker compose up -d --build   # or with docker
```

Runtime state (anime channels, whitelists) is stored in `data/config.json`.

## Everything is a component

`default.go` is the entire bot. Every feature is one entry in the config, so removing a line removes the feature.

```go
bot.New(bot.Config{
	Prefix:  ",",
	// background jobs
	OnReady: func(s *discordgo.Session, _ *discordgo.Ready) {
        // one line to start it
	    routines.StartAnimeNotify(s)
	},
	// LLM chat module: flip a flag, persona is just systemPrompt.md
	AI: bot.AIConfig{
	    Enabled: true,
	    SystemPrompt: systemPrompt,
	    ContextMessageCount: 10
	},
	// prefix commands
	Commands:      map[string]bot.CommandTemplate{
        // command_name -> handler
	    "ping": prefix.Ping
	},
	// slash commands
	SlashCommands: map[string]bot.SlashCommandEntry{
        // command_name -> handler
	    "coinflip": slash.CoinflipEntry
	},
})
```

### Adding a slash command: one file, one line

```go
// commands/slash/coinflip.go: the definition and the handler live together

// definition of the command
var CoinflipEntry = bot.SlashCommandEntry{
	Definition: &discordgo.ApplicationCommand{Name: "coinflip", Description: "Flips a coin."},
	Handler:    Coinflip,
}
// handler of the command
func Coinflip(s *discordgo.Session, i *discordgo.InteractionCreate) { /* respond */ }

// default.go: one line registration
"coinflip": slash.CoinflipEntry,
```

### Adding a prefix command: same idea

```go
// handler signature is all you need, args are already parsed for you
func Hello(s *discordgo.Session, m *discordgo.MessageCreate, args []string) {
	s.ChannelMessageSend(m.ChannelID, "hi")
}

// default.go: one line registration
"hello": prefix.Hello,
```

### Why it's pleasant to extend

- **No router, no boilerplate.** Dispatch is a map lookup, and slash commands are synced with Discord on startup. You never write registration code.
- **Handlers are plain functions.** No interfaces to implement and no base classes, so they're trivial to move between bots.
- **Features are isolated.** `ai/`, `commands/`, `routines/` and `config/` don't depend on each other. Delete a folder and its config line, and nothing else breaks.
- **Config-driven.** Set `GuildID` for instant guild-only slash commands, tweak `NamePatterns` for the AI trigger, or swap the persona by editing one markdown file.

## Layout

```
ai/        LLM client, key rotation
bot/       session, message routing, AI context builder
commands/  prefix/ and slash/ handlers
config/    JSON config with mutex + atomic writes
routines/  background jobs (anime notifier)
default.go bot wiring, systemPrompt.md = AI persona
```

## Create a bot

1. Go to https://discord.com/developers/applications.
2. Click "New Application" at the top right and name your bot.
3. Go to Bot in the left menu and enable the **Message Content** intent.
4. Scroll to "Token", press "Reset Token", and copy it into `.env` as `TOKEN`.

## Invite it to a server

1. Go to OAuth2 in the left menu.
2. In "OAuth2 URL Generator", check the `bot` scope.
3. Give it at least one permission, otherwise Discord doesn't create a role for the bot.
4. Open the generated URL to invite the bot.

## Roadmap

dynamic_vc, sticky messages, tags, code execution, karma, twitch notifications, starboard, channelstats
@@
