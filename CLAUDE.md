# uo-launch

Client **launch package** pages, served directly from this repo. Sibling of
`uo-onboarding`, same shape, different site and different Slack command.
Tracked in Linear **UNO-570**.

Hosted on **Vercel** (team `team-9698s-projects`).

**This repo is public**, matching `uo-onboarding`, which is public because
Vercel's Hobby plan will not deploy a private organisation repo. The team now
shows as Pro, so private may work here — if you switch it, confirm a deploy
still runs before assuming it does. Until then treat everything committed here
as published, and note that git history keeps every page ever added: removing
one from the site does not remove it from history.

## The two sites

|  | onboarding | launch |
|---|---|---|
| Repo | `uo-onboarding` | `uo-launch` (this one) |
| Publish dir | `deploy/uo-onboarding` | `deploy/uo-launch` |
| Slack command | `/newpageonboarding` | `/newpagelaunch` |
| Page state | `{ done, fields }` | `{ done, interest }` |

They are deliberately separate deployments. A launch page published into the
onboarding site is the mistake this repo exists to prevent — it happened with
`matureminds` before this split.

## URL layout — read before adding pages

- Pages live at `deploy/uo-launch/<client-slug>/index.html`.
- The publish directory is **`deploy/uo-launch`** (`vercel.json` ->
  `outputDirectory`), so a page is served at `<site>/<client-slug>/`. The
  `deploy/uo-launch/` prefix is **not** part of the public URL.
- Changing the publish directory relocates every existing client page and breaks
  links already sent to clients. Don't.

## Rules

- Each page is a self-contained Claude artifact bundled export (~480 KB, zero
  external requests). Never prettify, reformat, minify, or run a bundler over
  these files — they must stay byte-identical to what was exported.
- Never remove `.gitattributes` (`* -text`). It stops Windows checkouts from
  rewriting line endings and silently changing every deployed byte.
- No build step for the pages. Vercel serves everything under the publish
  directory as-is; do not add a build command. `package.json` exists only so
  Vercel installs the dependencies the functions need — it never runs over the
  client pages. Keep `package-lock.json` committed: `vercel.json` runs `npm ci`,
  which requires it.
- Client pages are served without auth and the slug is the only thing between a
  page and a stranger, so a page is only as private as its slug is unguessable.

## Functions

They live OUTSIDE the publish directory, so they deploy once and apply to every
client page — including pages published later, whose exports know nothing about
them. `/newpagelaunch` only ever writes under `deploy/uo-launch/<slug>/`, so a
publish cannot touch them.

Carried over from `uo-onboarding`, and the reasons still apply:

- **Named `GET`/`PUT` exports, never a default export.** A default export gets
  the Node `(req, res)` signature instead and a returned `Response` is silently
  discarded — the request then hangs until it times out.
- **`req.url` is relative on Vercel**, so `new URL(req.url)` throws. The base in
  `new URL(req.url, "http://localhost")` exists only to make it parse.
- **Preview isolation is by pathname, not by store.** Vercel Blob has one store
  per project, so state is written to `state/<production|preview>/<slug>.json`.
  `api/remove-page.ts` must build that path the same way or it deletes nothing.
- **Strong consistency is `useCache: false`** on `get()`. Without it a read
  right after a tick can be up to a minute stale, which reads as a failed save.

### What differs from onboarding

The launch page's state is `{ done, page, interest }`, not `{ done, fields }`.

- `interest` is the launch path the client picked. It **is** stored.
- `page` — which tab the client is on — is deliberately **not** stored. It is a
  per-person view position; syncing it would move the client's tab under them
  the moment a teammate opened the page. It stays in that browser's
  localStorage.
- The bundle ships with `const KEY = 'uo-launch-package-v1'`, the **same value
  in every launch export**. On localStorage that was invisible; against a shared
  store it would mean every client sharing one checklist. Both the API path and
  the localStorage key are therefore derived from the slug in the URL. Do not
  reintroduce a bare `KEY`.

Env vars needed on Vercel: `LIBRARY_PASSWORD`, `GITHUB_TOKEN`, plus the Blob
store connected to the project.
