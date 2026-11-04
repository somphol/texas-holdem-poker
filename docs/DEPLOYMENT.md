# Deployment

Hosted on **GitHub Pages** from the repository root. No build, no workflow: the committed
`index.html` on `main` is the game.

- Live URL: <https://somphol.github.io/texas-holdem-poker/>
- Repository: <https://github.com/somphol/texas-holdem-poker>
- Source branch: `main`, folder `/` (root)
- Entry file: `index.html`

## 1. Deploy a change

```bash
cd ~/Projects/texas-holdem-poker

# verify locally
python3 -m http.server 8000     # http://localhost:8000/
# play at least one full hand — see docs/TESTING.md

git add -A
git commit -m "…"
git push origin main
```

Confirm:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://somphol.github.io/texas-holdem-poker/
```

Pages updates within a minute or so. Hard-refresh if an old `index.html` appears; there is
no service worker.

## 2. Repository settings that matter

| Setting | Value | Why |
| --- | --- | --- |
| Pages source | branch `main`, `/ (root)` | the game is one root file |
| Custom domain | none | default `*.github.io` host |
| Enforce HTTPS | on | recommended; the game has no secure-only APIs but Pages is free |
| Actions | none | no workflow exists |

`.nojekyll` must remain committed.

## 3. Published files

The root is public: `index.html`, `screenshot.png`, `docs/social-card.png`, the Markdown
docs, `robots.txt`, `sitemap.xml`. No secrets, no keys, no environment variables, no
runtime configuration.

The game makes **no network requests**. If devtools shows one after a change, something
was added that breaks the single-file property.

## 4. Caching

Static files are served with ETags and short caching. The usual post-deploy symptom is a
stale `index.html` — the whole game is in that file, so there is nothing else to invalidate.
Use a private window to confirm a deploy.

## 5. Rollback

```bash
git revert <bad-commit>        # preferred
git push origin main
```

There is no staging environment; a revert is live immediately. Avoid force-pushing `main`
unless the bad commit must be removed from history.

## 6. Local versus production

`file://` works, including audio (after a gesture), because there are no fetches. The HTTP
path is what the release was verified on and is the closer match to Pages. Differences are
limited to origin-level browser behaviour; save data is not involved because the game
stores nothing.

## 7. Related deployment

The setup modal links to the separate **3D edition** at
<https://somphol.github.io/texas-holdem-3d/>. It has its own repository and docs; changing
this game does not deploy that one.
