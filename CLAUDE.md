# Japan Study — Japanese flashcard app

Single-file static web app: `index.html` (all HTML/CSS/JS inline) reads
`cards.json`, an array of `{ id, spanish, japanese, image }`. Images live in
`images/{id}.svg`. `japones_conversacional_200.csv` is the plain-text source
list kept in sync with `cards.json` (not read by the app itself). No build
step. Deployed via GitHub Pages at https://urijp.github.io/japan-study/,
remote `git@github.com:urijp/japan-study.git`, branch `main`.

The card count is **never hardcoded** anywhere (title, weighting, etc.) —
everything derives from `cards.length` at runtime. Keep it that way.

## When the user gives a list of new cards (spanish: japanese pairs)

Always do the full workflow below, not just an edit to `cards.json`:

1. **Check for duplicates** against the existing `cards.json` (match on the
   `japanese` field and the `spanish` field, case-insensitive). Skip any
   duplicates and tell the user which ones were skipped and why.
2. **Add each new card** with the next sequential `id` (max existing id + 1),
   appended to `cards.json`. Also append the raw `spanish,japanese` row to
   `japones_conversacional_200.csv` so the source list stays in sync.
3. **Always find and add a matching image for every new card** — never leave
   `image` empty. Source images from OpenMoji (CC BY-SA 4.0, free to
   redistribute):
   `https://cdn.jsdelivr.net/npm/openmoji@14.0.0/color/svg/{CODEPOINT}.svg`
   (uppercase hex codepoint, hyphen-joined for multi-codepoint sequences like
   flags; if the plain codepoint 404s, retry with `-FE0F` appended). Save to
   `images/{id}.svg`.
4. Pick the best-fitting emoji per concept. It's fine — expected, even — for
   multiple cards to share the same icon when they're thematically similar
   (e.g. all "entreno" variants, all karate-related cards). Matching the
   concept well matters more than every image being unique.
5. **Verify the downloaded image actually renders as intended**, not just
   that the HTTP request succeeded — OpenMoji has at least one mislabeled
   asset in this set (a codepoint that should be a name badge instead
   rendered as a fire icon). Load the app locally and visually confirm new
   cards before pushing.
6. Test locally before pushing: `python3 -m http.server 8080` in the project
   dir, open in a browser (Chrome DevTools MCP), switch to grid view, and
   confirm the new cards show the right text and image with no console
   errors.
7. **Commit and push to `main`** so GitHub Pages redeploys. Don't leave
   changes uncommitted — this is a deployed app, not just a local file.
