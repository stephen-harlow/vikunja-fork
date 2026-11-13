# Rebuilding the Vikunja fork

This is a fork of go-vikunja/vikunja that adds one feature: an email allowlist
for OpenID Connect sign-in (`auth.openid.providers.<id>.allowedemails`). The
single change is commit `e883ad2` on branch `allowlist`, touching only:

- `pkg/modules/auth/openid/openid.go`  (`AllowedEmails` field + `IsEmailAllowed` + gate in `getOrCreateUser`)
- `pkg/modules/auth/openid/providers.go` (`allowedemails` config parsing + `parseStringListField`)

Everything else is unmodified upstream.

## Upgrading to a new Vikunja release

1. Fetch upstream and check out the new tag, then re-apply the patch:

   ```sh
   cd ~/_Dev/_Dev2026/vikunja-fork
   git fetch --unshallow origin   # first upgrade only: this was a shallow clone
   git fetch origin --tags
   git checkout -B allowlist v<NEW-VERSION>
   git cherry-pick e883ad2        # resolve any conflicts in the two files above
   ```

2. Install the Go toolchain (user-local, no root) if not already present:

   ```sh
   # go.mod pins the exact version; install >= that. Currently go 1.27.0.
   curl -sL -o /tmp/go.tar.gz "https://go.dev/dl/go1.27.0.linux-amd64.tar.gz"
   mkdir -p ~/.local && tar -C ~/.local -xzf /tmp/go.tar.gz
   ```

3. Build the frontend (needed — it is embedded via `go:embed`):

   ```sh
   cd frontend
   pnpm install
   pnpm build          # vite build + workbox -> frontend/dist
   cd ..
   ```

4. Build the binary (stamp the fork version so it does not report `dev`):

   ```sh
   export PATH=$HOME/.local/go/bin:$PATH GOPATH=$HOME/go GOMODCACHE=$HOME/go/pkg/mod
   go build -ldflags '-X code.vikunja.io/api/pkg/version.Version=v<NEW-VERSION>-fork' -o /tmp/vikunja-build/vikunja .
   ```

   The version package is `pkg/version` (`code.vikunja.io/api/pkg/version`); the
   magefile normally stamps it via `git describe` plus `-X main.Tags`. The
   `-ldflags` above overrides `Version` directly and yields e.g.
   `Vikunja api version v2.8.0-fork` from `vikunja version`.

5. Deploy (service is a user unit running as `stephen`):

   ```sh
   BIN=~/.local/lib/todo/vikunja-<VERSION>/vikunja-v<NEW-VERSION>-linux-amd64
   systemctl --user stop todo-vikunja
   cp /tmp/vikunja-build/vikunja "$BIN"
   systemctl --user start todo-vikunja
   ```

   Point the service unit (`~/.config/systemd/user/todo-vikunja.service`) at the
   new version path, and update `~/_Dev/_Dev2026/Todo/scripts/fetch_release.py`
   `VERSION` if you change the pinned release.

## Verification

- `curl --unix-socket /run/user/1000/todo/vikunja.sock -s http://localhost/api/v2/info`
  must show `auth.openid_connect.enabled: true`, the `google` provider, and
  `allowed_emails` for stephen/elizabeth.
- A Google account NOT in `allowed_emails` must be refused with "email address
  is not allowed to sign in".

## Config (live, not in this repo)

- `~/.config/todo/config.yml` — `auth.openid.providers.google` (name, authurl,
  `allowedemails`) and `auth.local.enabled: false`.
- `~/.config/todo/todo.env` (0600) — `VIKUNJA_AUTH_OPENID_PROVIDERS_GOOGLE_CLIENTID`
  and `VIKUNJA_AUTH_OPENID_PROVIDERS_GOOGLE_CLIENTSECRET`.
- Google OAuth client redirect URIs (console.cloud.google.com):
  `https://todo.stephenharlow.com/auth/openid/google` and the trailing-slash variant.
