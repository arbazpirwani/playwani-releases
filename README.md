# playwani-releases

Public Android releases of Silk Road Chess. The website's Android link
(`https://silkroad.playwani.com/download/android`) redirects to
`https://github.com/arbazpirwani/playwani-releases/releases/latest/download/SilkChess.apk`, so it always serves the
newest release here.

## Android release workflow

`.github/workflows/android-release.yml` keeps that link on the version that is live, however it went live (a release by
script or a deploy by hand):

1. **Check, every 30 minutes** (about 10 seconds of runner time): it reads `version` from both hosts' `/api/health`
   (`silkroad.playwani.com` and `silk-road-chess.onrender.com`). If a host gives no answer or the two disagree (a deploy
   in progress), it stops and tries again at the next run. If this repo already has a release for that version, it stops:
   a release is never overwritten.
2. **Build**, only when the live version has no release here yet. It checks out the private repo `silk-road-chess` at the
   tag `v<BUILD>` (every scripted release makes one), or else at `main`, whose BUILD must then equal the live version.
   It runs the private repo's own `scripts/android-release.mjs` (the release-signed APK, with all its checks) on Java 21,
   Node 22 and the `pnpm` version from `packageManager`.
3. **Publish**: the private repo's `scripts/public-release.mjs` checks the APK and the notes again, waits until both
   hosts serve the version, creates the release as *latest* with `SilkChess-v<BUILD>.apk` and `SilkChess.apk`, then reads
   back the digests and the redirect. The player notes come from `docs/release-notes/v<BUILD>.md` in the private repo
   (at the tag, else on main) when that file passes the notes rule in `docs/public-release-notes.md`. Otherwise a
   generic note goes out ("Improvements and fixes from the latest update.") and the run shows a warning.

Because this repo is public, its run logs are public too. The workflow uploads no artifacts and prints no secrets. It has
no pull request or push trigger, so a fork can never reach the secrets. It deletes the key and the R8 `mapping.txt` when
the run ends. To read a crash report from an APK built here, run `node scripts/android-release.mjs` at the same tag on the
Mac: the mapping is written to `~/.config/playwani/mapping/`.

### The three secrets

| Secret | What it holds |
| --- | --- |
| `KEYSTORE_P12_BASE64` | the release key `silk-release.p12`, base64 |
| `KEYSTORE_PASSWORD` | the key's password (the `release-keystore-pass` file) |
| `PRIVATE_REPO_TOKEN` | a fine-grained token that can only read `silk-road-chess` |

Set them from your own terminal. The values go straight from the files to GitHub and are never pasted anywhere else:

```sh
export GH_TOKEN=$(gh auth token --user arbazpirwani)   # your personal account, not the work one
base64 -i ~/.config/playwani/silk-release.p12 | gh secret set KEYSTORE_P12_BASE64 --repo arbazpirwani/playwani-releases
gh secret set KEYSTORE_PASSWORD --repo arbazpirwani/playwani-releases < ~/.config/playwani/release-keystore-pass
gh secret set PRIVATE_REPO_TOKEN --repo arbazpirwani/playwani-releases   # it asks for the value: paste the token there
gh secret list --repo arbazpirwani/playwani-releases                      # names and dates only
unset GH_TOKEN
```

To make the token: GitHub, Settings, Developer settings, Personal access tokens, **Fine-grained tokens**, Generate new
token. Use these settings: Resource owner `arbazpirwani`. Repository access: **Only select repositories**,
`silk-road-chess`. Permissions: Repository, **Contents: Read-only** (Metadata read-only is added by itself). When the
token expires, the build stops with "PRIVATE_REPO_TOKEN cannot read". Make a new token and set the secret again.

### Run it by hand

Go to **Actions**, **Android release**, **Run workflow**:

- **dry_run ticked first.** It builds, signs and checks the APK of the live version and publishes nothing. This is the
  safe test after you set the secrets.
- **version**: leave it empty for the live version. Any other version can only be a dry run, because only the live
  version is published.
- Without dry_run, it publishes only when the live version has no release here yet.

The same from a terminal: `gh workflow run android-release.yml --repo arbazpirwani/playwani-releases -f dry_run=true`.
A script can ask for a run with `gh api repos/arbazpirwani/playwani-releases/dispatches -f event_type=release`.

### Good to know

- **One publisher wins.** If `push-release.mjs` or a run of `public-release.mjs` by hand publishes the version first,
  this workflow sees the release before its own publish step and ends green without publishing.
- **A failing build tries again every 30 minutes** until the cause is fixed or a newer version goes live. Each try sends
  a failure email. To pause it, open Actions, Android release, the "..." menu, Disable workflow.
- **GitHub disables scheduled workflows in a public repo after 60 days with no activity** and emails a warning first. To
  turn it back on, open Actions, Android release, Enable workflow.
