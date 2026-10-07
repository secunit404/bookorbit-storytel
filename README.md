# BookOrbit Storytel plugin

A [BookOrbit](https://github.com/bookorbit/bookorbit) request indexer plugin that finds audiobooks
on Storytel and downloads them with your own subscription.

Request a book the way you request any other. Storytel shows up as a source next to the rest; pick
its release and the plugin puts the book on your Storytel bookshelf, fetches the audio, and BookOrbit
imports it as an m4b with chapters, series and position, authors, narrators, publisher, date,
description, genre, language and cover.

## Requirements

- A Storytel subscription.
- [`ghcr.io/secunit404/bookorbit-patched`](https://github.com/secunit404/bookorbit-patched) for the
  m4b. Stock BookOrbit has no hook that turns a plain audio download into a tagged m4b, so on stock
  BookOrbit the plugin still works but imports the bare MP3 or M4A without chapters.

## Installing

1. Download the raw plugin file, not the GitHub page:

   ```sh
   curl -fL -o storytel-index.mjs https://raw.githubusercontent.com/secunit404/bookorbit-storytel/main/indexers/storytel/index.mjs
   ```

2. In BookOrbit, as a superuser: **Settings > System > Requests > Plugins > Install plugin**, and
   upload it. Or copy it to `/data/plugins/indexers/storytel/index.mjs` and restart.
3. **Set up** the Storytel source:
   - **API key**: your Storytel password. It is the only secret field a plugin gets.
   - **Storytel e-mail**: the account.
   - **Storytel store**: your market, e.g. `STHP-SE`, `STHP-NO`, `STHP-DK`, `STHP-FI`, `STHP-NL`, `STHP-DE`.
   - **Languages to search**: used when a request names no language, e.g. `sv, en`.
   - Then **Test**, which logs in once.

Updates are signed and offered by BookOrbit itself once version 0.2.0 or later is installed.

## Keeping the account safe

Storytel invalidates every session and emails the owner about "suspicious activity" when its audio
endpoint is hit too often, and Cloudflare blocks clients that look automated. The plugin is built
around that:

- **Search never logs in.** It uses Storytel's public catalogue, so browsing releases costs the
  account nothing.
- **The account is used only when a release is grabbed**, with one session reused for as long as
  Storytel accepts it.
- **Grabs run one at a time**, at most 10 per 24 hours and 60 seconds apart by default. Both are
  settings.
- **It backs off instead of retrying**: an hour after a Cloudflare block or rate limit, 15 minutes
  after a rejected password.
- **A book already on the bookshelf keeps its state**, so listening progress is never reset.

The counters live in memory and reset when BookOrbit restarts. This lowers the risk of the account
being flagged; it does not remove it, and automated downloading is unlikely to be what Storytel's
terms allow. Use your own subscription, for your own library.

## Trust

A plugin runs inside the BookOrbit process with that process's access: your database, library
files and encryption key. `indexers/storytel/index.mjs` is one dependency-free file so it can be
read before installing. It talks only to `storytel.com`, `storytel.net` and the cover and audio
CDNs Storytel links to, and only through BookOrbit's host fetch.

## Development

```sh
node indexers/storytel/verify.mjs
```

checks the plugin against saved Storytel responses and checks the signed manifest against the
source. To confirm the account endpoints against a real account, outside BookOrbit:

```sh
STORYTEL_EMAIL=you@example.com STORYTEL_PASSWORD='…' node indexers/storytel/try-account.mjs <consumableId> [--download]
```

The consumable id is the number at the end of a Storytel book URL. `--download` also builds the m4b
into `./storytel-test-<id>/`. One run is one login and one audio request.

After any change, bump `version` in `index.mjs` and re-sign. The private key stays outside the
repository:

```sh
BOOKORBIT_PLUGIN_SIGNING_KEY=~/.config/bookorbit-plugins/storytel-signing-key.pem node scripts/sign-update.mjs storytel
```

BookOrbit verifies the manifest's SHA-256 and Ed25519 signature against the key in the installed
plugin before offering an update.
