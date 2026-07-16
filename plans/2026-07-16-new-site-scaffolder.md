# Cement `new-site` CLI Scaffolder Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `npm run new-site` in the cement hub repo scaffolds a complete WordPress project — clones plugin + parent theme if missing, forks the child starter with an upstream remote, and applies every rename (theme header, text domain, handle prefix, stylelint token prefix, package name) from two prompts: project slug and token prefix.

**Architecture:** One CLI entry (`scripts/new-site.mjs`) that only prompts and orchestrates; all logic lives in two small libs — `scaffold.mjs` (pure string/JSON transforms + validators, fully unit-testable) and `git.mjs` (thin `execFileSync` wrappers, integration-tested against local fixture repos in a tmpdir, no network in tests).

**Tech Stack:** Node 18+ built-ins only — `node:readline/promises`, `node:child_process`, `node:fs`, `node:path`, `node:test`. Zero npm dependencies.

## Global Constraints

- Node `>=18` (uses `node:test`, `readline/promises`)
- No npm dependencies — built-ins only
- Repo: `reddstonebv/cement` (local clone: `~/Local Sites/cement-site`)
- Clone URLs are constants: `git@github.com:reddstonebv/cement-wp-plugin.git`, `git@github.com:reddstonebv/cement-wp-theme.git`, `git@github.com:reddstonebv/cement-wp-theme-child.git`
- Folder names are fixed by the theme system: plugin → `wp-cement-plugin`, parent → `wp-cement-theme`, child → `wp-cement-theme-<slug>` (child's `Template:` header depends on the parent folder name — never rename those)
- Tests are hermetic: git operations run against fixture repos created in `os.tmpdir()`, never against GitHub
- All git calls via `execFileSync` (array args — no shell interpolation of user input)
- Commit after every task

## File Structure

```
cement-site/
  package.json                 name, "type": "module", scripts: new-site, test
  scripts/
    new-site.mjs               CLI entry — prompts, calls lib, prints next steps
    lib/
      scaffold.mjs             validators + file transforms (pure functions)
      git.mjs                  clone / renameRemote / execFileSync wrappers
  test/
    scaffold.test.mjs          unit tests for every transform + validator
    git.test.mjs               integration tests against tmpdir fixture repos
    fixtures.mjs               helper that builds a fake child-starter repo in tmp
```

Reference: the real child starter files these transforms must handle —

`wp-cement-theme-child/style.css` (exact current header):

```css
/*
 * Theme Name:    Cement Child
 * Template:      wp-cement-theme
 * Version:       1.0.0
 * Author:        Reddstone
 * Author URI:    https://reddstone.nl
 * Text Domain:   cement-child
 * Project Slug:  cement-child
 */
```

`wp-cement-theme-child/functions.php` (relevant part):

```php
<?php
defined('ABSPATH') || exit;

add_action('wp_enqueue_scripts', function (): void {
    wp_enqueue_style(
        'cement-child-style',
        get_stylesheet_uri(),
        ['cement-style'],
        wp_get_theme()->get('Version')
    );
}, 20);
```

`wp-cement-theme-child/.stylelintrc.json` (shape per cement-tooling README):

```json
{
  "extends": ["@reddstonebv/cement-tooling/stylelint"],
  "rules": {
    "custom-property-pattern": ["^cmt-", { "message": "Token custom properties must be prefixed with 'cmt-'" }]
  }
}
```

---

### Task 1: Package + test runner scaffold

**Files:**
- Create: `package.json`
- Create: `test/scaffold.test.mjs` (smoke only, grows in later tasks)

**Interfaces:**
- Produces: `npm test` runs `node --test test/`; `npm run new-site` runs `node scripts/new-site.mjs`

- [ ] **Step 1: Write package.json**

```json
{
  "name": "@reddstonebv/cement",
  "private": true,
  "type": "module",
  "engines": { "node": ">=18" },
  "scripts": {
    "new-site": "node scripts/new-site.mjs",
    "test": "node --test test/"
  }
}
```

- [ ] **Step 2: Write a failing smoke test**

`test/scaffold.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { validateSlug } from '../scripts/lib/scaffold.mjs';

test('smoke: scaffold lib is importable', () => {
  assert.equal(typeof validateSlug, 'function');
});
```

- [ ] **Step 3: Run to verify it fails**

Run: `npm test`
Expected: FAIL — `Cannot find module '../scripts/lib/scaffold.mjs'`

- [ ] **Step 4: Create the empty lib**

`scripts/lib/scaffold.mjs`:

```js
export function validateSlug() {}
```

- [ ] **Step 5: Run to verify it passes**

Run: `npm test`
Expected: PASS (1 test)

- [ ] **Step 6: Commit**

```bash
git add package.json scripts/lib/scaffold.mjs test/scaffold.test.mjs
git commit -m "chore: package scaffold with node:test runner"
```

---

### Task 2: Validators

**Files:**
- Modify: `scripts/lib/scaffold.mjs`
- Test: `test/scaffold.test.mjs`

**Interfaces:**
- Produces: `validateSlug(s) -> string|null` and `validatePrefix(s) -> string|null` — return an error message string, or `null` when valid. Task 8's CLI loops a prompt until `null`.

- [ ] **Step 1: Write failing tests**

Append to `test/scaffold.test.mjs`:

```js
import { validatePrefix } from '../scripts/lib/scaffold.mjs';

test('slug: lowercase kebab ok', () => {
  assert.equal(validateSlug('rsr'), null);
  assert.equal(validateSlug('kasteel-beverweerd'), null);
});

test('slug: rejects uppercase, spaces, leading digit/dash, empty', () => {
  for (const bad of ['RSR', 'my site', '-x', '9lives', '', 'wp-cement-theme-x']) {
    assert.equal(typeof validateSlug(bad), 'string', `expected error for "${bad}"`);
  }
});

test('prefix: lowercase alnum ok, no dashes or empty', () => {
  assert.equal(validatePrefix('rsr'), null);
  assert.equal(validatePrefix('cmt2'), null);
  for (const bad of ['my-prefix', 'RSR', '', '9x']) {
    assert.equal(typeof validatePrefix(bad), 'string', `expected error for "${bad}"`);
  }
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test`
Expected: FAIL — validators return `undefined`

- [ ] **Step 3: Implement**

Replace `scripts/lib/scaffold.mjs` content:

```js
export function validateSlug(s) {
  if (!/^[a-z][a-z0-9-]*[a-z0-9]$|^[a-z]$/.test(s ?? '')) {
    return 'Slug must be lowercase kebab-case: letters, digits, dashes; starts with a letter.';
  }
  if (s.startsWith('wp-cement')) {
    return 'Slug is just the project part — "rsr", not "wp-cement-theme-rsr".';
  }
  return null;
}

export function validatePrefix(s) {
  if (!/^[a-z][a-z0-9]*$/.test(s ?? '')) {
    return 'Prefix must be lowercase letters/digits, starting with a letter (used as --<prefix>-*).';
  }
  return null;
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test`
Expected: PASS (4 tests)

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/scaffold.mjs test/scaffold.test.mjs
git commit -m "feat: slug and prefix validators"
```

---

### Task 3: style.css header transform

**Files:**
- Modify: `scripts/lib/scaffold.mjs`
- Test: `test/scaffold.test.mjs`

**Interfaces:**
- Produces: `transformStyleHeader(content, { displayName, slug }) -> string` — rewrites `Theme Name`, `Text Domain`, `Project Slug`, resets `Version` to `1.0.0`; preserves `Template`, `Author`, everything else byte-for-byte.

- [ ] **Step 1: Write failing test**

Append to `test/scaffold.test.mjs`:

```js
import { transformStyleHeader } from '../scripts/lib/scaffold.mjs';

const STYLE_HEADER = `/*
 * Theme Name:    Cement Child
 * Template:      wp-cement-theme
 * Version:       1.3.7
 * Author:        Reddstone
 * Author URI:    https://reddstone.nl
 * Text Domain:   cement-child
 * Project Slug:  cement-child
 */
`;

test('style header: rewrites name, domain, slug, resets version, keeps template', () => {
  const out = transformStyleHeader(STYLE_HEADER, { displayName: 'Cement RSR', slug: 'rsr' });
  assert.match(out, /Theme Name:\s+Cement RSR/);
  assert.match(out, /Text Domain:\s+cement-rsr/);
  assert.match(out, /Project Slug:\s+cement-rsr/);
  assert.match(out, /Version:\s+1\.0\.0/);
  assert.match(out, /Template:\s+wp-cement-theme/);
  assert.match(out, /Author:\s+Reddstone/);
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test`
Expected: FAIL — `transformStyleHeader` is not exported

- [ ] **Step 3: Implement**

Append to `scripts/lib/scaffold.mjs`:

```js
// Header lines look like " * Field Name:   value" — replace value, keep
// the line's original label + whitespace so the header stays aligned.
function setHeaderField(content, field, value) {
  const re = new RegExp(`^(\\s*\\*?\\s*${field}:\\s*).*$`, 'm');
  return content.replace(re, `$1${value}`);
}

export function transformStyleHeader(content, { displayName, slug }) {
  let out = content;
  out = setHeaderField(out, 'Theme Name', displayName);
  out = setHeaderField(out, 'Text Domain', `cement-${slug}`);
  out = setHeaderField(out, 'Project Slug', `cement-${slug}`);
  out = setHeaderField(out, 'Version', '1.0.0');
  return out;
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/scaffold.mjs test/scaffold.test.mjs
git commit -m "feat: style.css header transform"
```

---

### Task 4: functions.php handle transform

**Files:**
- Modify: `scripts/lib/scaffold.mjs`
- Test: `test/scaffold.test.mjs`

**Interfaces:**
- Produces: `transformFunctions(content, slug) -> string` — replaces the starter's own handle prefix `cement-child` with `cement-<slug>`. Parent handles (`cement-style`, `cement-tokens`, `cement-fonts`) must NOT change — they are dequeue/dependency targets owned by the parent theme.

- [ ] **Step 1: Write failing test**

Append to `test/scaffold.test.mjs`:

```js
import { transformFunctions } from '../scripts/lib/scaffold.mjs';

const FUNCTIONS = `<?php
wp_enqueue_style(
    'cement-child-style',
    get_stylesheet_uri(),
    ['cement-style'],
    wp_get_theme()->get('Version')
);
`;

test('functions: renames own handles, leaves parent handles alone', () => {
  const out = transformFunctions(FUNCTIONS, 'rsr');
  assert.match(out, /'cement-rsr-style'/);
  assert.match(out, /\['cement-style'\]/); // parent dep untouched
  assert.doesNotMatch(out, /cement-child/);
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test`
Expected: FAIL — not exported

- [ ] **Step 3: Implement**

Append to `scripts/lib/scaffold.mjs`:

```js
export function transformFunctions(content, slug) {
  // "cement-child" is the starter's own namespace; word-boundary match
  // so "cement-style" (parent handle) never matches.
  return content.replaceAll('cement-child', `cement-${slug}`);
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/scaffold.mjs test/scaffold.test.mjs
git commit -m "feat: functions.php handle transform"
```

---

### Task 5: .stylelintrc.json prefix transform

**Files:**
- Modify: `scripts/lib/scaffold.mjs`
- Test: `test/scaffold.test.mjs`

**Interfaces:**
- Produces: `transformStylelint(content, prefix) -> string` — JSON-parses, sets `rules["custom-property-pattern"]` to `["^<prefix>-", { message }]` (creating `rules` if absent), re-serializes with 2-space indent + trailing newline.

- [ ] **Step 1: Write failing tests**

Append to `test/scaffold.test.mjs`:

```js
import { transformStylelint } from '../scripts/lib/scaffold.mjs';

test('stylelint: sets custom-property-pattern for the prefix', () => {
  const input = JSON.stringify({
    extends: ['@reddstonebv/cement-tooling/stylelint'],
    rules: { 'custom-property-pattern': ['^cmt-', { message: 'old' }] },
  });
  const out = JSON.parse(transformStylelint(input, 'rsr'));
  assert.equal(out.rules['custom-property-pattern'][0], '^rsr-');
  assert.match(out.rules['custom-property-pattern'][1].message, /'rsr-'/);
  assert.deepEqual(out.extends, ['@reddstonebv/cement-tooling/stylelint']);
});

test('stylelint: creates rules block when missing', () => {
  const out = JSON.parse(transformStylelint('{"extends":["x"]}', 'abc'));
  assert.equal(out.rules['custom-property-pattern'][0], '^abc-');
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test`
Expected: FAIL — not exported

- [ ] **Step 3: Implement**

Append to `scripts/lib/scaffold.mjs`:

```js
export function transformStylelint(content, prefix) {
  const config = JSON.parse(content);
  config.rules ??= {};
  config.rules['custom-property-pattern'] = [
    `^${prefix}-`,
    { message: `Token custom properties must be prefixed with '${prefix}-'` },
  ];
  return JSON.stringify(config, null, 2) + '\n';
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/scaffold.mjs test/scaffold.test.mjs
git commit -m "feat: stylelint token-prefix transform"
```

---

### Task 6: package.json name transform + wp-content detection

**Files:**
- Modify: `scripts/lib/scaffold.mjs`
- Test: `test/scaffold.test.mjs`

**Interfaces:**
- Produces: `transformPackageJson(content, slug) -> string` — sets `name` to `@reddstonebv/cement-wp-theme-<slug>`, preserves everything else.
- Produces: `isWpContent(dir) -> boolean` — true when `dir` contains both `themes/` and `plugins/` directories (how the CLI validates the target path).

- [ ] **Step 1: Write failing tests**

Append to `test/scaffold.test.mjs`:

```js
import { mkdtempSync, mkdirSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { transformPackageJson, isWpContent } from '../scripts/lib/scaffold.mjs';

test('package.json: renames, preserves other fields', () => {
  const out = JSON.parse(
    transformPackageJson('{"name":"old","scripts":{"x":"y"}}', 'rsr')
  );
  assert.equal(out.name, '@reddstonebv/cement-wp-theme-rsr');
  assert.deepEqual(out.scripts, { x: 'y' });
});

test('isWpContent: true only with themes/ and plugins/', () => {
  const dir = mkdtempSync(join(tmpdir(), 'cement-test-'));
  assert.equal(isWpContent(dir), false);
  mkdirSync(join(dir, 'themes'));
  assert.equal(isWpContent(dir), false);
  mkdirSync(join(dir, 'plugins'));
  assert.equal(isWpContent(dir), true);
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test`
Expected: FAIL — not exported

- [ ] **Step 3: Implement**

Append to `scripts/lib/scaffold.mjs`:

```js
import { existsSync, statSync } from 'node:fs';
import { join } from 'node:path';

export function transformPackageJson(content, slug) {
  const pkg = JSON.parse(content);
  pkg.name = `@reddstonebv/cement-wp-theme-${slug}`;
  return JSON.stringify(pkg, null, 2) + '\n';
}

export function isWpContent(dir) {
  return ['themes', 'plugins'].every((d) => {
    const p = join(dir, d);
    return existsSync(p) && statSync(p).isDirectory();
  });
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/scaffold.mjs test/scaffold.test.mjs
git commit -m "feat: package.json transform + wp-content detection"
```

---

### Task 7: git helpers (fixture-tested)

**Files:**
- Create: `scripts/lib/git.mjs`
- Create: `test/fixtures.mjs`
- Test: `test/git.test.mjs`

**Interfaces:**
- Produces: `clone(url, dest)` — `git clone url dest`, throws on nonzero exit.
- Produces: `renameRemote(repoDir, from, to)` and `addRemote(repoDir, name, url)`.
- Produces: `remotes(repoDir) -> {name: url}` — parsed from `git remote -v` (fetch lines), used by tests and by the CLI's summary output.
- Consumes (tests): `makeFixtureChildRepo() -> path` from `test/fixtures.mjs` — a local git repo shaped like the child starter (style.css, functions.php, .stylelintrc.json, package.json), clonable by path. **No network anywhere in tests.**

- [ ] **Step 1: Write the fixture helper**

`test/fixtures.mjs`:

```js
import { mkdtempSync, writeFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { execFileSync } from 'node:child_process';

const STYLE = `/*
 * Theme Name:    Cement Child
 * Template:      wp-cement-theme
 * Version:       1.0.0
 * Author:        Reddstone
 * Text Domain:   cement-child
 * Project Slug:  cement-child
 */
`;

const FUNCTIONS = `<?php
defined('ABSPATH') || exit;
add_action('wp_enqueue_scripts', function (): void {
    wp_enqueue_style('cement-child-style', get_stylesheet_uri(), ['cement-style'], '1.0.0');
}, 20);
`;

export function makeFixtureChildRepo() {
  const dir = mkdtempSync(join(tmpdir(), 'cement-fixture-'));
  writeFileSync(join(dir, 'style.css'), STYLE);
  writeFileSync(join(dir, 'functions.php'), FUNCTIONS);
  writeFileSync(
    join(dir, '.stylelintrc.json'),
    JSON.stringify({ extends: ['@reddstonebv/cement-tooling/stylelint'] }, null, 2)
  );
  writeFileSync(join(dir, 'package.json'), JSON.stringify({ name: 'cement-child' }, null, 2));
  const git = (...args) => execFileSync('git', args, { cwd: dir });
  git('init', '-b', 'main');
  git('config', 'user.email', 'test@test');
  git('config', 'user.name', 'test');
  git('add', '-A');
  git('commit', '-m', 'fixture');
  return dir;
}
```

- [ ] **Step 2: Write failing tests**

`test/git.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { mkdtempSync, existsSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { clone, renameRemote, addRemote, remotes } from '../scripts/lib/git.mjs';
import { makeFixtureChildRepo } from './fixtures.mjs';

test('clone + remote juggling matches the fork recipe', () => {
  const fixture = makeFixtureChildRepo();
  const work = mkdtempSync(join(tmpdir(), 'cement-clone-'));
  const dest = join(work, 'wp-cement-theme-rsr');

  clone(fixture, dest);
  assert.ok(existsSync(join(dest, 'style.css')));

  renameRemote(dest, 'origin', 'upstream');
  addRemote(dest, 'origin', 'git@github.com:reddstonebv/cement-wp-theme-rsr.git');

  const r = remotes(dest);
  assert.equal(r.upstream, fixture);
  assert.equal(r.origin, 'git@github.com:reddstonebv/cement-wp-theme-rsr.git');
});
```

- [ ] **Step 3: Run to verify failure**

Run: `npm test`
Expected: FAIL — `Cannot find module '../scripts/lib/git.mjs'`

- [ ] **Step 4: Implement**

`scripts/lib/git.mjs`:

```js
import { execFileSync } from 'node:child_process';

function git(args, opts = {}) {
  return execFileSync('git', args, { stdio: ['ignore', 'pipe', 'inherit'], ...opts })
    .toString()
    .trim();
}

export function clone(url, dest) {
  git(['clone', url, dest]);
}

export function renameRemote(repoDir, from, to) {
  git(['remote', 'rename', from, to], { cwd: repoDir });
}

export function addRemote(repoDir, name, url) {
  git(['remote', 'add', name, url], { cwd: repoDir });
}

export function remotes(repoDir) {
  const out = git(['remote', '-v'], { cwd: repoDir });
  const map = {};
  for (const line of out.split('\n')) {
    const m = line.match(/^(\S+)\s+(\S+)\s+\(fetch\)$/);
    if (m) map[m[1]] = m[2];
  }
  return map;
}
```

- [ ] **Step 5: Run to verify pass**

Run: `npm test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add scripts/lib/git.mjs test/fixtures.mjs test/git.test.mjs
git commit -m "feat: git helpers with hermetic fixture tests"
```

---

### Task 8: scaffoldProject orchestrator (still no prompts)

**Files:**
- Modify: `scripts/lib/scaffold.mjs`
- Test: `test/scaffold.test.mjs`

**Interfaces:**
- Consumes: all transforms (Tasks 3–6), git helpers (Task 7).
- Produces: `scaffoldProject({ wpContent, slug, prefix, displayName, urls }) -> { themeDir, cloned: string[] }` — the single function the CLI calls. `urls` is `{ plugin, parent, child }` (injectable so tests pass fixture paths). Clones plugin/parent only if their folders are missing (records which in `cloned`), clones child to `wp-cement-theme-<slug>`, renames origin→upstream, applies all four file transforms in place. Does **not** add the new origin remote (URL isn't knowable until the GitHub repo exists — the CLI prints that as a next step).

- [ ] **Step 1: Write failing test**

Append to `test/scaffold.test.mjs`:

```js
import { readFileSync, mkdtempSync as mkdtemp2, mkdirSync as mkdir2 } from 'node:fs';
import { scaffoldProject } from '../scripts/lib/scaffold.mjs';
import { makeFixtureChildRepo } from './fixtures.mjs';
import { remotes } from '../scripts/lib/git.mjs';

test('scaffoldProject: end-to-end against fixtures', () => {
  const child = makeFixtureChildRepo();
  const plugin = makeFixtureChildRepo(); // shape irrelevant, just clonable
  const parent = makeFixtureChildRepo();

  const wpContent = mkdtemp2(join(tmpdir(), 'cement-wpc-'));
  mkdir2(join(wpContent, 'themes'));
  mkdir2(join(wpContent, 'plugins'));

  const result = scaffoldProject({
    wpContent,
    slug: 'rsr',
    prefix: 'rsr',
    displayName: 'Cement RSR',
    urls: { plugin, parent, child },
  });

  assert.equal(result.themeDir, join(wpContent, 'themes', 'wp-cement-theme-rsr'));
  assert.deepEqual(result.cloned.sort(), ['parent', 'plugin']);

  const style = readFileSync(join(result.themeDir, 'style.css'), 'utf8');
  assert.match(style, /Theme Name:\s+Cement RSR/);
  assert.match(style, /Text Domain:\s+cement-rsr/);

  const fn = readFileSync(join(result.themeDir, 'functions.php'), 'utf8');
  assert.doesNotMatch(fn, /cement-child/);

  const lint = JSON.parse(readFileSync(join(result.themeDir, '.stylelintrc.json'), 'utf8'));
  assert.equal(lint.rules['custom-property-pattern'][0], '^rsr-');

  assert.equal(remotes(result.themeDir).upstream, child);

  // second run with plugin/parent present: clones nothing extra
  const again = scaffoldProject({
    wpContent, slug: 'other', prefix: 'oth', displayName: 'Other', urls: { plugin, parent, child },
  });
  assert.deepEqual(again.cloned, []);
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test`
Expected: FAIL — `scaffoldProject` not exported

- [ ] **Step 3: Implement**

Append to `scripts/lib/scaffold.mjs`:

```js
import { readFileSync, writeFileSync } from 'node:fs';
import { clone, renameRemote } from './git.mjs';

function transformFile(path, fn) {
  if (!existsSync(path)) return;
  writeFileSync(path, fn(readFileSync(path, 'utf8')));
}

export function scaffoldProject({ wpContent, slug, prefix, displayName, urls }) {
  const cloned = [];

  const pluginDir = join(wpContent, 'plugins', 'wp-cement-plugin');
  if (!existsSync(pluginDir)) {
    clone(urls.plugin, pluginDir);
    cloned.push('plugin');
  }

  const parentDir = join(wpContent, 'themes', 'wp-cement-theme');
  if (!existsSync(parentDir)) {
    clone(urls.parent, parentDir);
    cloned.push('parent');
  }

  const themeDir = join(wpContent, 'themes', `wp-cement-theme-${slug}`);
  if (existsSync(themeDir)) {
    throw new Error(`${themeDir} already exists — pick another slug or remove it.`);
  }
  clone(urls.child, themeDir);
  renameRemote(themeDir, 'origin', 'upstream');

  transformFile(join(themeDir, 'style.css'), (c) =>
    transformStyleHeader(c, { displayName, slug }));
  transformFile(join(themeDir, 'functions.php'), (c) =>
    transformFunctions(c, slug));
  transformFile(join(themeDir, '.stylelintrc.json'), (c) =>
    transformStylelint(c, prefix));
  transformFile(join(themeDir, 'package.json'), (c) =>
    transformPackageJson(c, slug));

  return { themeDir, cloned };
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test`
Expected: PASS (all tests)

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/scaffold.mjs test/scaffold.test.mjs
git commit -m "feat: scaffoldProject orchestrator"
```

---

### Task 9: CLI entry with prompts

**Files:**
- Create: `scripts/new-site.mjs`

**Interfaces:**
- Consumes: `scaffoldProject`, `validateSlug`, `validatePrefix`, `isWpContent` (scaffold.mjs).
- Produces: the `npm run new-site` UX. Prompts (in order): wp-content path (default: cwd if it passes `isWpContent`), slug, prefix (default: slug), display name (default: `Cement <slug uppercased>`). Prints created/skipped repos and exact next steps. No unit test — this file is prompt-and-print only; everything it calls is already tested. Verified manually in Step 2.

- [ ] **Step 1: Implement**

`scripts/new-site.mjs`:

```js
#!/usr/bin/env node
/**
 * new-site — scaffold a Cement WordPress project.
 * Prompts for slug + token prefix, clones plugin/parent if missing,
 * forks the child starter (origin→upstream) and applies all renames.
 */
import { createInterface } from 'node:readline/promises';
import { resolve } from 'node:path';
import {
  validateSlug, validatePrefix, isWpContent, scaffoldProject,
} from './lib/scaffold.mjs';

const URLS = {
  plugin: 'git@github.com:reddstonebv/cement-wp-plugin.git',
  parent: 'git@github.com:reddstonebv/cement-wp-theme.git',
  child: 'git@github.com:reddstonebv/cement-wp-theme-child.git',
};

const rl = createInterface({ input: process.stdin, output: process.stdout });

async function ask(question, { fallback = '', validate = () => null } = {}) {
  for (;;) {
    const suffix = fallback ? ` (${fallback})` : '';
    const answer = (await rl.question(`${question}${suffix}: `)).trim() || fallback;
    const error = validate(answer);
    if (!error) return answer;
    console.error(`  ✗ ${error}`);
  }
}

const cwd = process.cwd();
const wpContent = resolve(await ask('Path to wp-content', {
  fallback: isWpContent(cwd) ? cwd : '',
  validate: (p) => (p && isWpContent(resolve(p)) ? null
    : 'Not a wp-content directory (needs themes/ and plugins/).'),
}));

const slug = await ask('Project slug (e.g. rsr)', { validate: validateSlug });
const prefix = await ask('Token prefix', { fallback: slug, validate: validatePrefix });
const displayName = await ask('Theme display name', {
  fallback: `Cement ${slug.toUpperCase()}`,
});
rl.close();

const { themeDir, cloned } = scaffoldProject({ wpContent, slug, prefix, displayName, urls: URLS });

console.log(`\n✓ Theme scaffolded at ${themeDir}`);
for (const repo of ['plugin', 'parent']) {
  console.log(cloned.includes(repo) ? `✓ Cloned ${repo}` : `• ${repo} already present — skipped`);
}
console.log(`
Next steps:
  1. Create the GitHub repo: reddstonebv/cement-wp-theme-${slug}
  2. cd ${themeDir}
     git add -A && git commit -m "chore: bootstrap cement-wp-theme-${slug}"
     git remote add origin git@github.com:reddstonebv/cement-wp-theme-${slug}.git
     git push -u origin main
  3. WP admin → activate the Cement plugin and the "${displayName}" theme
  4. Settings → Permalinks → Post name
  5. npm install   (pre-commit hooks; needs /tmp/cement-tooling symlink)

Sync starter updates later:  git fetch upstream && git merge upstream/main
`);
```

- [ ] **Step 2: Manual smoke test against fixtures**

```bash
mkdir -p /tmp/fake-wpc/themes /tmp/fake-wpc/plugins
npm run new-site
# answer: /tmp/fake-wpc · testproj · (enter) · (enter)
```

Expected: clones fail against the real GitHub URLs only if SSH is unavailable — on the dev machine this completes; verify `/tmp/fake-wpc/themes/wp-cement-theme-testproj/style.css` says `Theme Name:    Cement TESTPROJ` and `git remote -v` shows `upstream`. Clean up: `rm -rf /tmp/fake-wpc`.

- [ ] **Step 3: Run the full test suite once more**

Run: `npm test`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add scripts/new-site.mjs
git commit -m "feat: new-site CLI entry"
```

---

### Task 10: Documentation

**Files:**
- Modify: `README.md` (cement repo — "Starting a new project" section)
- Modify: `docs/index.html` (WordPress path, step 3)

**Interfaces:**
- Consumes: the CLI UX from Task 9 (command + prompt names must match exactly).

- [ ] **Step 1: README — add the CLI as the primary route**

In `README.md`, replace the "Starting a new project" intro paragraph with:

```markdown
## Starting a new project

Fastest route — clone this repo and run the scaffolder:

```bash
git clone git@github.com:reddstonebv/cement.git && cd cement
npm run new-site
```

It asks for the wp-content path, a project slug and a token prefix, then
clones the plugin + parent theme (if missing), forks the child starter with
an `upstream` remote, and applies every rename automatically.

Prefer manual? Open the **[setup guide](https://reddstonebv.github.io/cement/)** —
it walks the same steps by hand, starting with the one question that
decides everything else: **WordPress or static?**
```

- [ ] **Step 2: Pages guide — reference the CLI in the fork step**

In `docs/index.html`, inside the WordPress path's step 3 (`Fork the child starter`), add after the `<pre>` block:

```html
<div class="note">Or skip steps 2–3 entirely: clone <a href="https://github.com/reddstonebv/cement">reddstonebv/cement</a> and run <code>npm run new-site</code> — it clones what's missing, forks the starter and applies the whole rename checklist from two prompts.</div>
```

- [ ] **Step 3: Commit + push**

```bash
git add README.md docs/index.html
git commit -m "docs: document npm run new-site as the primary bootstrap route"
git push
```

---

## Deliberately out of scope (YAGNI)

- **Static-site scaffolding** — the static path is two steps (a `<link>` tag and an npm install); no automation needed yet.
- **GitHub repo auto-creation** (`gh repo create`) — needs `gh` auth; printed as a next step instead. Add later if it grates.
- **In-WP setup page** — the plugin detecting a missing child theme and pointing at this command. Separate plan, separate repo, only after the CLI proves itself.
- **Windows support** — team is macOS; `execFileSync('git', …)` works there anyway, just untested.

## Self-review notes

- Spec coverage: prompts for prefix ✓, auto-namespacing (4 file transforms) ✓, places plugin + parent ✓, `npm run new-site` ✓, docs updated ✓. The "page on site.local" idea is explicitly parked in out-of-scope.
- Type consistency: `scaffoldProject` signature matches between Task 8 (definition) and Task 9 (call site); `remotes()` used in Tasks 7–8 tests only.
- Hermeticity: only Task 9's manual smoke test touches the network; all `node --test` runs use tmpdir fixtures.
