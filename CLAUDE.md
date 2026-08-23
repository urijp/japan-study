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

## Sentence Game (`sentence-game.pretty.v4.json`)

A separate mode, now the default view, that shows 3–5 cards drawn from
`sentence-game.pretty.v4.json` for the player to assemble into a spoken Japanese
sentence. It is **not synced with `cards.json`** (a deliberate, separate decision
made when the mode was built) — treat it as its own dataset.

**Architecture, and why it matters for edits:** the generator (`SentenceEngine`,
inside `index.html` between `// --- SENTENCE ENGINE START` and `// --- SENTENCE
ENGINE END ---`) is 100% generic — it has **zero hardcoded category names,
vocabulary, or particle literals**, only schema knowledge (field names like
`from`, `fixed`, `accepts`, `tags`, `particle`, `useForm`, `requiresArg(s)`,
`verbFilter*`, `tenseFrom`). This means **adding vocabulary is almost always a
pure JSON edit** — `index.html` only needs a touch when a genuinely new
grammatical *role* needs a caption (see "Captions" below).

### The data shape

- `vocab`: a dict of category → array of `{es, jp, ...}`. Current categories:
  `time, place, person, food, drink, transport, clothing, body, object, media,
  sport, activity, language, direction, demonstrative, adjective, state,
  adverb, connector, locationWord, question, number, social, concept,
  predicate, marker`. Extra fields by category: `time`/`state` have `timeRef`;
  `place`/`object`/`media`/`social` have `tags`; `adjective` has `adjClass`
  (`"i"`/`"na"`) and `accepts` (which noun categories it can modify).
- `verbs`: array of `{es, forms: {nonpastAffirmative, pastAffirmative,
  nonpastNegative, pastNegative}, args?: [{role, particle, accepts}], want?,
  notes?}`. All four `forms` are required. `args` says what nouns can fill the
  verb's object/location/destination/etc. and with which particle.
- `templates`: array of `{name, slots: [...], order: [...], literals?: {...},
  tenseFrom?, useForm?, subjectParticle?, verbFilter?/verbFilterHas?,
  example}`. A slot is `{role, from: [...] | "verb.args.X" | fixed: "..."}`,
  optionally `optional`, `particle`, `allowed`, `recommended`,
  `recommendedTags`, `requiresArg(s)`.

### The golden rule: literals are glue, never meaning

`literals` may **only** hold pure grammatical connectors with no independent
meaning of their own (`wa`, `ga`, `o`, `ni`, `to`, `no`). Anything with real
lexical content — a predicate like "gustar", the copula "desu", a question
marker, an interrogative word — **must be a card**, i.e. a real slot
referencing a `vocab` item (usually via `fixed` when it's always the same word
for that template). This was violated and fixed several times already (`suki
desu`, `shusshin desu`, bare `desu`, the `ka?` marker) — don't reintroduce it.
The one narrow, deliberate exception: `demonstrative-price-question`'s `ikura`
item has its `jp` merged to `"ikura desu"` because its own `es` gloss
(`"cuánto cuesta"`) is already a complete verb phrase that implies the
predicate — that reasoning does **not** extend to bare pronouns like `qué`/
`dónde`, which must keep a separate copula card.

### Adding new vocabulary — the workflow

1. **Pick the right category.** Nouns go in their semantic category
   (`food`, `place`, `person`, etc.); proper names go in `person` alongside
   the role words already there (`padre`, `amigo`, ...). Adjectives go in
   `adjective` with `adjClass` (`i`/`na`) and an `accepts` array listing every
   noun category it can grammatically modify (check `rules.adjectives` in the
   file for the compatibility rule). Prefer an existing category over
   inventing one.
2. **Only invent a new category when nothing fits AND the word would
   otherwise leak into an unrelated pool.** `predicate` and `marker` exist as
   their own categories specifically so `suki desu`/`shusshin desu`/`desu`/
   `ka?` are reachable *only* via the specific `fixed` slots that need them,
   not as random draws in `because-state`/`state-with-adverb` (which pool
   from `state`/`adjective`). If you add a category, add a matching
   one-line `categorySchema` entry too.
3. **Check for `jp` collisions across every category** before adding — `jp`
   must be globally unique (the engine indexes vocab by `jp` alone). A quick
   `python3 -c "..."` scan over `vocab.values()` is enough.
4. **Tags** (`tags: [...]`) exist so templates can narrow a pool with
   `category:tag` selectors (e.g. `media:readable`, `object:buyable`,
   `place:city`). If the new item fits a tag already used elsewhere in that
   category, add it.
5. **New verbs** need all four `forms`. Add `args` (role/particle/accepts) if
   the verb takes an object/location/destination/etc., and a `want` field
   only if the desiderative form makes sense — the UI already derives a
   "querer + infinitivo" label automatically from `useForm`, no code change
   needed. Add `notes` (not `args`) for verbs generation should treat as
   limited (see `decir`).
6. **Never add a `fixed`-slot value that isn't a real vocab item.** Every
   `fixed: "..."` must resolve by `jp` lookup — that's how the engine gives it
   a real Spanish gloss and reveal text.

### Suggest new templates when it's warranted

After adding vocabulary — especially a new verb with an argument shape no
current template uses, a new adjective/tag combination, or enough related
nouns to justify a themed round — **proactively suggest one or two concrete
new templates** if genuinely interesting, rather than only appending vocab
silently. Propose the exact `slots`/`order`/`example` and ask before adding,
since a new template changes round-generation variety and card counts.

### Captions (`ROLE_LABELS` in `index.html`)

Only touch this when a template introduces a genuinely new grammatical
*role* name. Conventions learned the hard way:

- Caption the role's **grammatical function**, not the specific word drawn
  (`adverb → 'adverbio'`, not `'cuánto'` — that assumed only degree adverbs,
  but the category also holds frequency/temporal/manner words).
- If a role is **always resolved via the same single `fixed` value** with no
  pool variety (e.g. `questionMarker`, or `kono` in
  `demonstrative-price-question`), consider **omitting the caption entirely**
  — it can only ever repeat the word or force an awkward invented label.
- Don't share one role name (and caption) across two grammatically different
  usages. `kore` (a demonstrative *pronoun* acting as subject) and `kono` (a
  demonstrative *adjective* modifying a noun) used to share the
  `demonstrative` role/caption until that was split into `pronoun` (`'sujeto'`)
  vs `demonstrative` (no caption) — one caption can't honestly describe both.

### Verification

After any edit to `sentence-game.pretty.v4.json`, re-run the self-test before
pushing:

```bash
python3 -c "
import re
html = open('index.html', encoding='utf-8').read()
m = re.search(r'// --- SENTENCE ENGINE START.*?// --- SENTENCE ENGINE END ---', html, re.S)
open('/tmp/engine_block.js','w').write(m.group(0))
"
node -e "
const fs = require('fs');
let src = fs.readFileSync('/tmp/engine_block.js', 'utf8');
src = src.replace(/^\/\/ --- SENTENCE ENGINE START.*\n/, '').replace(/\/\/ --- SENTENCE ENGINE END ---\$/, '');
global.SentenceEngine = eval(src.replace(/^\s*const SentenceEngine = /, ''));
const raw = JSON.parse(fs.readFileSync('sentence-game.pretty.v4.json', 'utf8'));
const engine = SentenceEngine.build(raw);
console.log('integrity:', JSON.stringify(engine.integrity));
console.log(engine.selfTest(5000).summary);
"
```

Expect `disabledTemplates: []`, `warnings: []`, and `5000/5000 rounds ok, ...,
0 templates never hit, 0 disabled`. (A single sporadic "generated invalid
round for X" warning that still resolves to N/N ok is a known, pre-existing,
self-healing retry unrelated to any specific edit — not a regression.) Then
spot-check `meta.sentenceText` for whatever templates you touched, load the
app locally (same server/browser workflow as above) with `?selftest=1` for an
in-browser confirmation, and visually check the new/changed cards in Sentence
Game mode before committing and pushing.
