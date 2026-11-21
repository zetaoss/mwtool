# mwcli

**Unofficial but universal MediaWiki CLI. Like `gh`, but for MediaWiki.**

`mw` is MediaWiki on the command line. It logs in to any MediaWiki wiki, reads and edits pages and makes Action API requests.

```sh
mw auth login --wiki mediawiki.org
mw auth status
mw page view 'Main Page'
mw page edit 'Sandbox' --summary 'Fix typo' --file page.wiki
mw api action=query meta=userinfo uiprop=rights
```

mwcli works with any MediaWiki wiki, Wikimedia wikis included. [a Wikimedia CLI](https://gitlab.wikimedia.org/repos/releng/cli), a separate tool for Wikimedia sites and services, also installs a command named `mw`; if you use both, run that one as `wm`, a name it supports.

## Installation

```sh
winget install mw                                # Windows (mwcli.mwcli)
go install github.com/mwcli/mwcli/cmd/mw@latest  # with Go
```

Archives for Linux, macOS and Windows are on the [releases](https://github.com/mwcli/mwcli/releases) page.

Or build from source with `make` (the binary is `bin/mw`) and install it with `make install` (`prefix=/usr/local` by default). Building requires Go 1.27 or newer. `mw --version` shows the version; releases are listed at <https://github.com/mwcli/mwcli/releases>.

## Choosing a wiki

Commands that talk to a wiki take `--wiki WIKI`: a host name such as `mediawiki.org` or an interwiki prefix such as `wikipedia`. Without it, the wiki is chosen in this order:

1. `MW_WIKI`
2. the host of the last `mw auth login`
3. `en.wikipedia.org`

mw finds the host's `api.php` itself: it tries `https://HOST/w/api.php`, then `https://HOST/api.php`, then the `EditURI` link of the main page, follows redirects (so `mediawiki.org` reaches `www.mediawiki.org`) and checks that the answer comes from MediaWiki. Like the kubectl discovery cache, the result and the site information are kept in `discovery/HOST/siteinfo.json` in the cache directory (`$XDG_CACHE_HOME/mw`, else `%LocalAppData%\mw` on Windows, else `~/.cache/mw`) for 24 hours, and dropped when the API answers 404. Deleting the directory is always safe.

Accounts are stored under the host of the API, so `mediawiki.org` and `www.mediawiki.org` are one wiki.

`--wiki` and `MW_WIKI` take the interwiki prefixes of MediaWiki's default [interwiki list](https://github.com/wikimedia/mediawiki/blob/master/maintenance/interwiki.list) that point to MediaWiki wikis, such as `wikipedia` (en.wikipedia.org), `commons`, `wikidata`, `mw` (www.mediawiki.org) and `metawikimedia`; `mw help interwiki` lists all 21. A name with a dot is always a host name.

```sh
mw page view --wiki commons 'Main Page'
```

`--api URL` names the endpoint directly; `mw auth login --api URL` stores it in `hosts.yml` for that host, and then no discovery is done for it.

## Authentication

`mw auth login` prefers an owner-only OAuth 2 access token. Create one on the wiki at `Special:OAuthConsumerRegistration/propose/oauth2` ("This consumer is for use only by …") with the grants the tool needs, such as basic and editpage. Owner-only tokens do not expire; revoke them at `Special:OAuthManageMyGrants`. `--with-token` reads the token from standard input. `--username` logs in with a password through the `clientlogin` flow instead, read from `MW_PASSWORD` or a no-echo prompt; extra steps such as two-factor authentication are not supported.

Credentials are stored per host and account in `hosts.yml` in the configuration directory, with owner-only permissions. Treat this file as a credential. The directory is `MW_CONFIG_DIR`, else `$XDG_CONFIG_HOME/mw`, else `%AppData%\mw` on Windows, else `~/.config/mw`.

- `mw auth status` checks every stored account and shows its API, auth method, masked token, which of edit, create, move, delete and protect it may do, and its rights. `-t` shows the whole token.
  - The token shows only its last 4 characters; one of 8 characters or fewer is masked entirely.
  - MediaWiki's OAuth 2 tokens are JWTs. When the token carries the claims, its client (first 8 characters of `aud`), issue date (`iat`) and OAuth grants (`scopes`) are shown too. They are decoded for display only, not verified.
- `mw auth switch` changes the active account of a host; `mw auth logout` removes one; `mw auth token` prints a token.
- `MW_TOKEN` uses that access token instead of stored credentials, for scripts and CI.

To check one page, including protection and blocks, ask the wiki with `mw api`. `"actions":{"edit":true}` means the edit would succeed; add `intestactionsdetail=full` to get the reasons when it would not:

```sh
mw api action=query prop=info intestactions=edit titles=Sandbox
```

## API requests

`mw api` sends `key=value` arguments (or `-f key=value`) to `api.php` and prints the response, indented on a terminal. `-F key=@file` reads a value from a file (`@-` for standard input). `format=json` and `formatversion=2` are added unless given. With `-X POST` and no `token`, a CSRF token is fetched and added.

## Exit status

- 0: success
- 1: the command failed, or the API reported an error
- 2: wrong usage
- 4: a login is required

See [docs/command.md](docs/command.md) for the command tree.

## Releasing

`VERSION` holds the version of the next release. A pull request that changes it is a release pull request: once merged to `main`, the Release workflow tags `vX.Y.Z` from `VERSION` and GoReleaser publishes the GitHub release with binaries for Linux, macOS and Windows. A prerelease such as `0.3.0-rc.1` is published as a GitHub prerelease and never marked latest (and `go install …@latest` skips it); a final version is marked latest only when it is the highest published one. The release is uploaded as a draft and published last; if the run fails after tagging, re-running it (or running the workflow by hand on `main`; other branches are ignored) resumes from the tag. CI checks that `VERSION` is a semantic version greater than the latest tag by SemVer precedence. Nobody pushes tags by hand.
