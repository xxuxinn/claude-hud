# Fork Guide — `xxuxinn/claude-hud`

How this personal fork came to be, how to install it on a new device, and how to
pull in upstream releases without losing the customizations.

- **Fork:** https://github.com/xxuxinn/claude-hud
- **Upstream:** https://github.com/jarrodwatts/claude-hud (MIT license)
- **Forked at:** upstream `0.4.2`, first fork commit `8ddb2df` (2026-07-15)

---

## 1. What this fork changes

**Technical:** three deliberate divergences from upstream, all small and localized.
Everything else is untouched upstream code.

**In plain terms:** this copy rearranges the HUD the way I like it and turns the
full display on by default — so a fresh install looks right immediately, with no
settings file needed.

| # | Change | File | Detail |
|---|--------|------|--------|
| 1 | Session tokens on line 1 | `src/render/index.ts` | In `render()`'s expanded branch, the `Tokens X (in/out/cache)` segment is joined onto `lines[0]` (the `[model] │ project git │ …` identity line) with a ` │ ` separator, instead of `lines.push(...)` at the bottom. Falls back to its own line only if no other lines rendered. |
| 2 | Counts + tools share a line | `src/config.ts` | `DEFAULT_MERGE_GROUPS` gains `['environment', 'tools']`, so the `2 CLAUDE.md \| 23 rules \| …` line and the `✓ Bash ×12 \| ✓ Edit ×4 …` line merge when they fit the terminal width (they stack automatically when too narrow — upstream's `canCombine` logic handles this). |
| 3 | Full display by default | `src/config.ts` | Seven `DEFAULT_CONFIG.display` toggles flipped `false → true`: `showConfigCounts`, `showSpeed`, `showTools`, `showAgents`, `showTodos`, `showSessionName`, `showSessionTokens`. A device with **no** `config.json` now renders the complete HUD. A local `config.json` still overrides everything, as upstream designed. |

Test suite adjustments that ride along (they assert the new intended behavior):

- `tests/config.test.js` — the `showSessionName` default assertion flipped to `true`.
- `tests/fixtures/expected/render-basic.txt` — integration snapshot regenerated via
  `npm run test:update-snapshots`.

**Technical:** `dist/` is committed in this repo (upstream convention — CI compiles
it automatically; in the fork you rebuild it manually with `npm run build`, which
`npm test` already does). Never commit `src/` changes without a matching `dist/`.

**In plain terms:** the repo ships both the source code and a pre-compiled copy.
The compiled copy must be regenerated whenever the source changes — running the
tests does this for you as a side effect.

---

## 2. Install on a new device

**Technical:** the plugin installs from this fork as a Claude Code marketplace; the
statusline is wired manually in `~/.claude/settings.json` because the command
includes a custom clock helper and the `CLAUDE_HUD_ALLOW_EXTRA_CMD` gate.

**In plain terms:** two slash commands fetch the plugin; then you paste one block
into your settings file and drop in a tiny clock script. Five minutes, start to
finish.

### Step 1 — Prerequisites

- Claude Code installed and signed in.
- [bun](https://bun.sh) at `~/.bun/bin/bun` (the statusline runs the plugin's
  TypeScript directly with bun). If bun lives elsewhere, adjust the path in the
  statusline command below — or swap the `exec` to `node "${plugin_dir}dist/index.js"`
  to avoid bun entirely.

### Step 2 — Add the marketplace and install

Inside any Claude Code session:

```
/plugin marketplace add xxuxinn/claude-hud
/plugin install claude-hud@claude-hud
```

(Equivalent from a shell: `claude plugin marketplace add xxuxinn/claude-hud` then
`claude plugin install claude-hud@claude-hud`.)

### Step 3 — Create the clock helper

Save as `~/.claude/statusline-clock.sh` and `chmod +x` it:

```bash
#!/usr/bin/env bash
# claude-hud extra-cmd helper: emits the current wall-clock time as a HUD label.
# Wired via the statusLine `--extra-cmd` arg in settings.json; renders on the first
# (project) row after the git branch. Replaces the session-duration element
# (display.showDuration:false). Output must be JSON: {"label":"..."}.
printf '{"label":"🕐 %s"}\n' "$(date +"%Y-%m-%d %H:%M")"
```

### Step 4 — Wire the statusline

Add this `statusLine` block to `~/.claude/settings.json` (top level). It is one
long command string — copy it exactly:

```json
"statusLine": {
  "type": "command",
  "command": "bash -c 'export CLAUDE_HUD_ALLOW_EXTRA_CMD=1; cols=$(stty size </dev/tty 2>/dev/null | awk '\"'\"'{print $2}'\"'\"'); export COLUMNS=$(( ${cols:-120} > 4 ? ${cols:-120} - 4 : 1 )); plugin_dir=$(ls -d \"${CLAUDE_CONFIG_DIR:-$HOME/.claude}\"/plugins/cache/*/claude-hud/*/ 2>/dev/null | awk -F/ '\"'\"'{ print $(NF-1) \"\\t\" $(0) }'\"'\"' | grep -E '\"'\"'^[0-9]+\\.[0-9]+\\.[0-9]+[[:space:]]'\"'\"' | sort -t. -k1,1n -k2,2n -k3,3n -k4,4n | tail -1 | cut -f2-); exec \"$HOME/.bun/bin/bun\" --env-file /dev/null \"${plugin_dir}src/index.ts\" --extra-cmd \"$HOME/.claude/statusline-clock.sh\"'",
  "refreshInterval": 60
}
```

**Technical:** the command measures the terminal width, locates the
highest-numbered installed plugin version under `plugins/cache/*/claude-hud/*/`,
and executes it with bun. `CLAUDE_HUD_ALLOW_EXTRA_CMD=1` is **required since
upstream 0.4.x** — without it, `--extra-cmd` is silently ignored (a security gate,
because that flag executes an arbitrary command) and the clock disappears.

**In plain terms:** the long command finds the newest installed copy of the plugin
and runs it, with a switch flipped that permits the little clock script to run.
Forget the switch and everything works except the clock silently vanishes.

### Step 5 — Restart and verify

Restart Claude Code. The first HUD line should look like:

```
[Fable 5] │ my-project git:(main) │ 🕐 2026-07-15 08:41 │ Tokens 6.2M (in: 110, out: 41k, cache: 6.1M)
```

No `config.json` is needed — the fork's defaults already produce this layout. Only
create `~/.claude/plugins/claude-hud/config.json` for per-device overrides
(language, colors, extra toggles); it wins over the baked-in defaults.

---

## 3. When upstream ships a new version

**Technical:** Claude Code's update checks now point at the **fork**, not upstream —
`claude plugin update claude-hud` only ever fetches what the fork's `main` contains.
Upstream releases must be merged into the fork manually; nothing breaks if you skip
one, you just don't get its features.

**In plain terms:** your devices drink from your own well. New upstream releases
don't reach you until you pour them in — which also means an upstream release can
never silently wipe out your customizations again.

### Step 1 — Notice that upstream released

```bash
gh api repos/jarrodwatts/claude-hud/tags --jq '.[0].name'   # latest upstream tag
gh api repos/xxuxinn/claude-hud/contents/package.json --jq '.content' | base64 -d | grep version
```

Or just skim upstream's `CHANGELOG.md` occasionally.

### Step 2 — Clone the fork and merge upstream

```bash
gh repo clone xxuxinn/claude-hud   # use HTTPS if SSH port 22 is blocked (see Troubleshooting)
cd claude-hud
git remote add upstream https://github.com/jarrodwatts/claude-hud.git
git fetch upstream
git merge upstream/main
```

**Do not use `gh repo sync`** — the fork's `main` has diverged from upstream, and
`gh repo sync --force` would discard the fork commits.

### Step 3 — Resolve conflicts (if any)

Only three regions can conflict, matching the three changes in section 1:

1. **`src/render/index.ts`** — the session-tokens block in `render()`. Keep the fork
   shape (join onto `lines[0]`), adopt any surrounding upstream changes:
   ```ts
   // Session token usage (cumulative), appended to the identity line
   if (ctx.config?.display?.showSessionTokens) {
     const sessionTokensLine = renderSessionTokensLine(ctx);
     if (sessionTokensLine) {
       if (lines.length > 0) {
         lines[0] = `${lines[0]} │ ${sessionTokensLine}`;
       } else {
         lines.push(sessionTokensLine);
       }
     }
   }
   ```
2. **`src/config.ts`** — keep `['environment', 'tools']` in `DEFAULT_MERGE_GROUPS`
   and the seven `true` display defaults, adopt everything else from upstream.
3. **`dist/*` and the snapshot** — never hand-merge compiled output. Take either
   side (`git checkout --theirs dist/`), finish the merge, then regenerate:
   the build in the next step overwrites `dist/` correctly.

### Step 4 — Rebuild, test, verify

```bash
npm install
npm test                          # runs `tsc` (rebuilds dist/) + full suite
npm run test:update-snapshots     # ONLY if failures are the fork's intended layout diffs
```

Then eyeball real output (any Claude Code status JSON works as stdin):

```bash
echo '{"model":{"display_name":"Test"},"transcript_path":"/path/to/some-session.jsonl","cwd":"'$PWD'"}' \
  | CLAUDE_HUD_ALLOW_EXTRA_CMD=1 COLUMNS=200 node dist/index.js
```

Check: tokens on line 1, counts+tools merged.

### Step 5 — Push and roll out to devices

```bash
git add -A && git commit -m "chore: merge upstream vX.Y.Z" && git push origin main
```

On each device: `claude plugin update claude-hud`, then restart Claude Code.
If upstream's new version changed statusline requirements (like 0.4.x's
`CLAUDE_HUD_ALLOW_EXTRA_CMD` gate did), read their CHANGELOG for the migration
note and adjust `settings.json` accordingly.

### Fallback — reapply by hand

If a merge ever gets ugly, reset to upstream and reapply from scratch; the full
recipe is the three edits in section 1 plus the two test adjustments. That is the
entire fork.

```bash
git checkout -B main upstream/main   # start clean from upstream
# reapply section-1 edits 1–3, fix the two tests, npm test, push --force-with-lease
```

---

## 4. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Clock (`🕐`) missing from line 1 | `CLAUDE_HUD_ALLOW_EXTRA_CMD` not set (0.4.x+ gate) | Ensure the statusline command starts with `export CLAUDE_HUD_ALLOW_EXTRA_CMD=1;` |
| `git clone`/`push` fails: `Connection closed by 198.18.x.x port 22` | Clash TUN (fake-IP range) intercepts SSH | Use HTTPS remotes; run `gh auth setup-git` once so git authenticates via `gh` |
| HUD shows an old layout after update | Statusline picks the highest version dir; stale caches are harmless but a restart is needed for plugin (non-statusline) parts | Restart Claude Code; optionally delete old dirs under `~/.claude/plugins/cache/claude-hud/claude-hud/` |
| Statusline blank after uninstall/reinstall | No version dir matched the `ls -d .../claude-hud/*/` glob | `claude plugin install claude-hud@claude-hud`, restart |
