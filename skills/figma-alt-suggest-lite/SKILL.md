---
name: figma-alt-suggest-lite
description: >-
  Lightweight version of the "figma-alt-suggest" skill. Walks the current
  canvas selection (FRAME, SECTION, COMPONENT, or COMPONENT_SET), finds every
  node with an IMAGE fill — including generically named ones like
  `Rectangle`/`Image 1` — captures each one's actual rendered content through
  the Plugin API, and proposes a specific Japanese alt-text description for
  each, distinguishing meaningful images (needs a real description) from
  purely decorative ones (icons beside labelled text, background textures).
  Read-only — nothing is written to the file. Figma Design Agent edition. For
  the full skill that also writes the approved suggestions as Dev Mode
  annotations after confirmation, use `figma-alt-suggest`.
---

# Figma Alt-Text Suggestions (Lite)

**Ver:** ver.202608111500

For every image placed in the selection, look at what it actually shows and propose a specific Japanese alt-text description. This is a read-only check-and-report skill — it never writes anything to the file.

**Output language:** All user-facing output (tables, report) is Japanese. If the user writes in another language, follow their language instead.

**Environment:** This edition runs inside Figma's agent environment. The target comes from the current canvas selection, and all reads go through the Plugin API script execution tool (e.g. `evaluate_script`) — no URLs, no external MCP tools.

## Why this exists

Alt text usually gets decided (or skipped) at implementation time, by whoever is coding the page — and they're often guessing from a filename or a layer name, not looking at the actual image. This skill moves that judgment call earlier, into the design review, where the person looking at the file can see exactly what each image contains. This lite edition stops at the report: it looks at every image and proposes text for it, but never touches the file. For confirmation-gated write of the approved suggestions as Dev Mode annotations, use the full `figma-alt-suggest` skill.

## Scope

**What this does:** finds every node with a visible `IMAGE` fill anywhere in the selection subtree, captures its rendered content, and proposes alt text (or flags it as decorative with no text needed).

**What this does NOT do:** vector icons/illustrations with no `IMAGE` fill (pure `VECTOR`/`BOOLEAN_OPERATION` shapes), write anything to the file (no annotations, no fills, no layer names — for that, use the full `figma-alt-suggest` skill), judge WCAG conformance beyond the alt-text content itself (that's `figma-contrast-check`'s territory for color), or accessibility notes beyond ALT text (landmarks, ARIA patterns — that's `figma-annotate`'s territory).

**Target node types:** FRAME, SECTION, COMPONENT, or COMPONENT_SET. Any other node type selected: ask the user to select a valid target.

---

## Step 1: Resolve the target

Use the current canvas selection. If nothing is selected, ask the user to select the frame/section/component to check. If the selection type isn't FRAME/SECTION/COMPONENT/COMPONENT_SET, ask for a valid selection instead.

## Step 2: Find image-fill nodes

Run via the Plugin API script tool. Walk the node tree depth-first from the selection root — do not skip `INSTANCE` subtrees, since a photo dropped inside a card/product component instance is exactly the kind of image this skill needs to catch. `fills` reads normally on nodes inside an `INSTANCE`, whether or not the fill is an instance-level override — don't assume it's inaccessible just because the node lives inside an instance.

**Check the selection root itself, not just its descendants.** A descendants-only traversal (e.g. `node.findAll(...)`, which only inspects children) never inspects the root node it was called on — so a selected `FRAME` that has the `IMAGE` fill directly on itself (a common pattern: a hero section frame with a photo as its own background fill, heading text as a child on top) gets silently skipped entirely. Check `selection.fills` explicitly before or alongside walking `selection.findAll(...)` — don't rely on the descendants walk alone to cover the root.

For every node, check `node.fills` (an array; skip if not applicable to the node type) for any entry where `type === 'IMAGE'` and the fill itself is `visible !== false`. Collect: `id`, `name`, `type`, absolute width/height (for the decorative-size heuristic in Step 3), the parent chain (for the report's layer path and for reading nearby context), and the fill's `imageHash` — access it as `node.fills.find(f => f.type === 'IMAGE').imageHash`, not `node.imageHash` (that property doesn't exist on the node itself).

Don't filter by node name — the point of this scan is that names like `Rectangle`, `Image 1`, or `画像` tell you nothing about what's actually in the fill. A node's name is never a substitute for looking at it.

If zero image-fill nodes are found, say so and stop.

**Group by `imageHash` before doing anything else.** This is a cross-node comparison over the whole collected list, not something to check one node at a time — do it here, once, right after collection, not inside the per-node loop in Step 3. For `INSTANCE` subtrees specifically, also check whether the image is defined on the shared `mainComponent` (not an instance-level override); if so, treat repeated instances the same as an `imageHash` match. Any group with 2+ members is a duplicate group — carry that grouping into Step 3 so only one member of each group needs a full visual description; the rest are handled by reference (see Step 3, point 4).

## Step 3: Capture and classify each image

For each collected node (for duplicate groups from Step 2, only the first member needs points 1–3 below — the rest are handled at point 4):

1. **Capture its rendered content** — use `await node.screenshot()` to get an inline image you can actually view. Don't reach for `node.exportAsync()` here: in this environment it returns raw bytes only, with no way to visually inspect them — using it silently defeats the whole point of this skill, since you'd be back to guessing instead of looking, with no way to notice you'd done so.
   - **Batch size:** don't call `screenshot()` on every collected node inside one giant loop in a single script execution — large batches have been observed to drop results silently (a 9-image run returned only 7 on the first pass, with no error). Capture in small batches (roughly 5–7 at a time). If a batch still comes back short, don't just retry the same size — drop to a smaller batch (e.g. half) for the retry, since a short batch is itself a sign the current size is too large for this file.
   - **Verifying a batch came back complete:** `screenshot()` doesn't return a countable value your script can check — the only way to tell if a batch is complete is to count the inline images that actually appear in the tool output against the batch size you requested. Treat this as a visual/cognitive check you make after each batch, not something you can assert programmatically. If the count is short, re-request only the missing nodes — don't treat a dropped result as "no image found" or silently skip it from the report.
2. **Read a bit of surrounding context** for judgment, not for guessing content:
   - Nearby visible `TEXT` nodes — for spotting redundant icon+label pairs (icon + adjacent label with the same meaning).
   - **Sibling** `TEXT` nodes under the same parent, for that case.
   - **Descendant** `TEXT` nodes of the same frame, when the image is applied as that `FRAME`'s own fill rather than living on a separate child node — a hero image is very often the frame's *background fill* with the heading as its *child*, not its sibling. Missing this distinction is the single most common way to wrongly classify a text-overlaid hero as "needs a description" when it's actually decorative.
   - The parent/section name and the node's absolute size relative to the frame it sits in.
3. **Classify:**
   - **装飾（Decorative)** — propose when the image is icon-scale (both dimensions roughly ≤ 64px) and sits next to a `TEXT` node whose visible content already conveys the same meaning (e.g. an arrow icon beside a "続きを見る" label), OR the image is a large background/texture layer with other content composited on top of it (full-bleed photo behind a hero heading, repeating pattern fill). Don't decide this from size or position alone — confirm the visual content actually looks decorative (a full-bleed *product photo* behind a heading is not decorative just because something sits on top of it; a soft gradient or texture is).
   - **要説明文（Needs a description)** — everything else: product shots, portraits, illustrations that carry information the surrounding text doesn't already state.

   For "needs a description" images, write a concise, specific Japanese description of what's actually depicted — what's shown, not "〜の画像" or "〜を写した写真" framing. For "decorative" images, don't force a description; note *why* it's decorative (echoes adjacent label text / pure background texture) instead.
4. **For the remaining members of a duplicate group** (identified in Step 2), don't repeat points 1–3 — reference the first member instead and flag the duplication in the report: reuse the same description for all of them, treat repeats after the first as decorative (`alt=""`), or write a distinguishing caption if the surrounding context differs enough to warrant one. Let the user decide which; don't pick silently.

If a node's fill can't be captured after a retry (screenshot error, extremely large node), mark it `⚠️ 取得失敗` in the report and note the reason — don't guess a description for content you couldn't actually see.

## Step 4: Report (compact by default)

Output the compact summary first — don't volunteer the full per-image table up front:

---

## 画像ALTテキスト提案レポート

**対象:** [ノード名] (`nodeId`)

### サマリー

| 分類 | 件数 |
|------|------|
| 要説明文 | N |
| 装飾（説明文不要） | N |
| ⚠️ 取得失敗 | N |

（合計 N 件。同一画像の重複が M 組見つかった場合はここに1行追記: 「うち同一画像の重複が M 組あります（詳細は下記）」）

### 提案の一部（先頭3件）

1. **[レイヤーパス]** — 提案ALT: 「[具体的な説明文]」
2. …
3. …

（気になった点があれば一言 — 例: ファイル名と中身が食い違っている画像があった場合、同一画像の重複があった場合など）

全件の一覧を見たい場合は「詳細を見せて」と伝えてください。

---

**On request** (「詳細を見せて」等), output the full table:

| # | レイヤー | サイズ | 分類 | 提案ALT |
|---|---------|--------|------|---------|
| 1 | Hero > BackgroundPhoto | 1440×760 | 要説明文 | "湯気の立つ土鍋に出汁を注いでいる、和食店の厨房の様子" |
| 2 | CTA > ArrowIcon | 24×24 | 装飾 | （説明文不要 — 隣接する「続きを見る」ラベルと意味が重複） |
| 3 | Footer > MapThumbnail | 320×180 | 要説明文 | "店舗周辺の地図。最寄駅から徒歩5分の位置に赤いピンで店舗が示されている" |

For a node flagged as a duplicate in Step 3, note it inline in the 提案ALT cell instead of adding a new column — e.g. `"（#3と同一画像。#3の説明を流用するか、装飾扱いにするか要検討）"` — so the duplication stays visible without restructuring the table per-run.

これは提案のみで、Figmaファイルへの変更は行っていません。この内容をDev Modeのアノテーションとして書き込みたい場合は、フルバージョンの `figma-alt-suggest` を実行してください。

---

## Error handling

- No selection, or selection isn't FRAME/SECTION/COMPONENT/COMPONENT_SET: ask the user to select a valid target
- Zero image-fill nodes found: say so plainly and stop — this is a valid, common outcome, not an error
- Script execution / screenshot error on a specific node: mark that row `⚠️ 取得失敗` and continue with the rest, don't abort the whole scan
- Very large trees: if a single walk is too slow or errors, walk top-level sections one at a time and merge the results
