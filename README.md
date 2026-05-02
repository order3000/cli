# `order3000` — agent-friendly CLI for the order3000 platform

A strict, schema-validated command-line interface backed by the
[`order3000`](../order3000-nodejs) Node.js SDK. Every command is also a
deterministically-shaped JSON contract suitable for agents (Claude, CI bots,
internal scripts) to call without writing TypeScript.

The CLI's input schemas are generated end-to-end from the order3000 backend's
`ts-rest` contracts → `openapi.json` → SDK Zod schemas, so the validation a
caller sees on `order3000 orders place` matches what the backend will accept
byte-for-byte.

---

## Install via Homebrew (macOS — Apple Silicon or Intel)

This is the recommended path. The Homebrew binary bundles the Node runtime,
so you don't need to install Node yourself.

```sh
# 1. Add the order3000 tap (one-time setup)
brew tap order3000/tap

# 2. Install the CLI
brew install order3000

# 3. Verify
order3000 --version
```

That's it. From here, `order3000` is on your `$PATH` like any other Homebrew
binary. To upgrade to the latest release later: `brew upgrade order3000`.

> **Linux Homebrew users**: the same two commands work. `brew tap …` then
> `brew install order3000`.

### What if `--keychain` shows "OS keychain unavailable"?

The Homebrew binary cannot bundle native modules across architectures, so
the optional OS-keychain integration (`auth login --keychain`) is only
available when you install via npm:

```sh
pnpm i -g @order3000/cli
```

The default file-based credential storage works perfectly with the brew
binary — you only need the npm install if you specifically want secrets
in macOS Keychain / Linux libsecret.

---

## First run: `auth login`

```sh
order3000 auth login --profile default
```

You'll be prompted for the robot account's `clientId` and `clientSecret`.
The CLI verifies them against `/auth/token` before saving, so a typo
fails fast instead of poisoning the profile.

Profile lives at `~/.config/order3000/profiles.json` (file mode `0600`).
Cached JWT lives at `~/.cache/order3000/tokens.json` (also `0600`).

You can also drive the CLI entirely from environment variables — useful in
CI, scripts, and agent contexts:

```sh
export ORDER3000_CLIENT_ID=robot_xxxxxx
export ORDER3000_CLIENT_SECRET=secret_xxxxxx
export ORDER3000_BASE_URL=https://order3000.com/api
order3000 auth whoami --json
```

Resolution order (highest priority first): flags → env vars → profile in
`profiles.json` → OS keychain (npm install only).

---

## Quick start

```sh
# Discover the command tree
order3000 --help
order3000 orders --help
order3000 orders get --help

# Read the strict JSON Schema for any command (input + output)
order3000 orders place --schema | jq

# Run a read-only command
order3000 orders getLatest --session-id <session-id> --json

# Run a mutating command (in non-TTY mode you must confirm with --yes)
order3000 orders place \
  --basket-id <id> \
  --session-id <id> \
  --payment-method cash \
  --fulfillment-type pickup \
  --yes
```

Output adapts automatically: pretty when stdout is a TTY, JSON otherwise.
Force JSON anywhere with `--json`.

---

## Command surface

| Group | Commands |
|---|---|
| `auth` | `login`, `logout`, `list`, `use`, `whoami` |
| `orders` | `getLatest`, `get`, `place`, `confirm` |
| `baskets` | `get`, `add`, `remove`, `getTotals`, `abandon`, `setDeclaredReturns` |
| `events` | `list`, `get`, `getByAccessor`, `place` |
| `menus` | `get`, `getByAccessor`, `getBundles`, `getSubscriptions` |
| `areas` | `list`, `getByAccessor`, `getAvailability`, `submitBookingRequest` |
| `reservations` | `make`, `get` |
| `paymentIntents` | `create`, `createDelta`, `reduceDelta` |
| `subscriptions` | `get`, `getByAccessor`, `subscribe`, `cancel`, `getCredits` |
| `teamOrders` | `create`, `get`, `join`, `submit`, `cancel`, `extendDeadline`, `setFulfillmentAt`, `setDiningLocation`, `setDiningTime`, `setOrderLeadSettled`, `autoExtendFulfillmentIfExpired`, `switchFulfillment`, `updateDeliveryAddress` |
| `system` | `ping` |

Every command honours these universal flags:

| Flag | Purpose |
|---|---|
| `--profile <name>` | Pick a non-default profile from `profiles.json` |
| `--client-id` / `--client-secret` / `--base-url` / `--venue-id` | Override credentials per-call |
| `--json`, `--format pretty\|json\|jsonl` | Output format (TTY-aware default) |
| `--schema` | Print the command's input + output JSON Schema and exit |
| `--yes` / `-y` | Confirm a mutating command in non-TTY mode |

Exit codes: `0` ok · `1` user error · `2` auth · `3` network · `4` server · `99` unexpected.

---

## Exit codes — agents please read

JSON-mode output goes to **stdout**; errors and human-friendly diagnostics
go to **stderr**. Always:

- Branch on the **exit code**, never on the presence of stderr text.
- Parse stdout as JSON when `--json` is passed (or when stdout is not a TTY,
  which is the agent default).
- Use `--schema` once per command at startup to discover the expected input
  and output shape; cache it.

---

## From source (this monorepo)

```sh
# Build the SDK + CLI from a local checkout
pnpm --filter order3000 build
pnpm --filter @order3000/cli build

# Run the just-built binary
node apps/order3000-cli/dist/cli.js --help

# Or compile to a single-file native binary (this Mac)
pnpm --filter @order3000/cli build:binaries darwin-arm64
./apps/order3000-cli/dist/binaries/order3000-darwin-arm64 --help
```

---

## Schemas as a public contract

Every command's input + output JSON Schema is committed under
`schemas/v1/` and validated against generated output on every build
(`pnpm build:schemas:check`). Drift between the CLI's published contract
and the SDK's runtime is a build-time error, not a runtime surprise.

The schemas themselves derive from the order3000 backend's `ts-rest`
contracts (in `apps/order3000/src/app/api/openapi/contracts/`), which are
in turn derived from the Drizzle schema via `drizzle-zod`. There is one
source of truth.
