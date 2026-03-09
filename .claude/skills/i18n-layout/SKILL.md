---
name: i18n-layout
description: This skill should be used when the user asks to translate UI text layers in Sketch, apply multilingual layout rules, handle overflow after translation, fix text overlapping after localization, adapt spacing for Portuguese/Spanish/Indonesian/German/Russian or other languages, or mentions "i18n", "多语言排版", "翻译后溢出", "字体缩小", "副标题重叠", "俄语", "俄文". Also activates when working with Sketch artboards targeting specific regional markets (South America, Southeast Asia, Europe, Russia/CIS).
version: 1.2.0
---

# ax-i18n-pro — UI Multilingual Layout Optimizer

Translating UI text in Sketch is never just a find-replace. Each language has different expansion rates, word-break rules, and visual rhythm. This skill enforces a 4-rule pipeline that guarantees pixel-safe, visually balanced results every time.

## Activation Scope

Apply this skill whenever:
- Replacing text layers in Sketch with a translated version
- Detecting overflow after translation
- Repositioning title/subtitle groups post-translation
- Adapting layouts for **Portuguese (BR/PT)**, **Spanish (ES/LATAM)**, **Indonesian (ID)**, **German (DE)**, **French (FR)**, **Russian (RU)**

---

## Rule 1 — Dynamic Overflow Detection (防溢出检测)

After every text replacement, **always** measure the resulting bounding box.

```js
// After setting layer.text = newText:
const f = layer.frame;
const pf = layer.parent.frame;
const overflowX = (f.x + f.width) > pf.width + 2;
const overflowY = (f.y + f.height) > pf.height + 2;
```

**Fix priority order:**
1. If `fixedWidth = true` → **set `fixedWidth = false` first** and re-measure; the text may already fit on one line without any font change
2. If still overflowing as `fixedWidth = false` (width > artboard bounds) → **accept 2-line layout** by restoring `fixedWidth = true`; do NOT reduce font size just to avoid wrapping — cross-frame consistency matters more
3. Reduce `fontSize` proportionally only when 2-line layout is unacceptable. Use **direct ratio calculation** (not a while-loop):
   ```js
   // ✅ CORRECT — one-shot calculation, avoids Sketch API float non-convergence
   const targetFs = Math.max(currentFs * (maxWidth / currentWidth), minFontSize);
   layer.style.fontSize = targetFs;
   ```
4. If still overflowing after size reduction → insert `\n` at the nearest semantic pause (comma, preposition, conjunction)
5. Never truncate text with ellipsis unless explicitly requested

**Chinese note**: Chinese characters are visually more compact than Latin at the same `fontSize`. A 9-char Chinese title at 29px routinely fits where a 22-char English title at the same size wrapped. Do not pre-emptively reduce font size for Chinese translations.

**Language expansion budgets** (vs. English baseline):

| Language | Avg expansion | Max single-word | Min font floor |
|----------|--------------|-----------------|----------------|
| Portuguese (BR) | +15–25% | +40% (compound verbs) | 20px |
| Spanish (LATAM) | +20–30% | +50% (articles+noun) | 20px |
| Indonesian | +10–20% | +60% (prefixed verbs) | 18px |
| German | +25–40% | +80% (compounds) | 16px |
| French | +15–25% | +35% | 20px |
| Russian | +20–35% | +50% (prefixed verbs) | 20px |

**Russian vs Chinese**: Cyrillic chars at the same `fontSize` are ~1.7× wider than Chinese chars (Cyrillic ~15–17px/char at 29px; Chinese ~29px/char but each character is more visually compact). A 7-char Chinese subtitle that fits at 16px will almost certainly overflow when replaced with a 15–20 char Russian equivalent. Always re-measure after CJK → Russian replacement.

See [references/language-expansion-rates.md](references/language-expansion-rates.md) for detailed tables.

---

## Rule 2 — Relative Layout & Anti-collision (相对布局防重叠)

When a title wraps to a second line, its group height increases. **Never** leave subtitle position hardcoded.

**Standard two-line title group spec:**
- Group: `y: 18`, `height: 101`
- Subtitle y: `title_frame.y + title_frame.height + 12`

**Standard single-line title group spec:**
- Group: `y: 36`, `height: 66`
- Subtitle y: `title_frame.y + title_frame.height + 12`

**Collision resolution algorithm:**
```
1. Measure title height after text set
2. If title.height > single_line_threshold (≈35px):
     move entire group to y:18, set group.height = 101
   Else:
     keep group at y:36, set group.height = 66
3. Set subtitle.y = title.frame.y + title.frame.height + 12
4. Verify subtitle bottom < artboard.height - safe_margin (≥ 8px)
```

**Critical**: Adjust the *group* position, not individual text layers within the group. Compressing inter-element gaps destroys visual rhythm.

**Title group identification**: When iterating `ab.layers` to find the title group, **only consider direct children with `y < 150`**. Artboards containing phone screen mockups often have inner UI groups (e.g. "Group 8" at `y=185`) that also contain large-font text — these must not be mistaken for the artboard title group. Safe filter:

```js
ab.layers.forEach(l => {
  if ((l.type === 'Group' || l.isFrame) && l.layers && l.frame.y < 150) {
    const hasTitle = l.layers.some(c => c.type === 'Text' && c.style.fontSize >= 20);
    if (hasTitle && !titleGroup) titleGroup = l;
  }
});
```

---

## Rule 2b — Cross-frame Font Size Consistency (跨画板字号一致性)

When batch-translating multiple artboards, all same-role text layers (e.g., all H1 titles) **must end up at the same `fontSize`**.

**Problem**: Per-frame overflow handling can silently produce different font sizes across artboards (e.g., Frame 2 title at 22px, Frame 3 at 27px, Frame 4 at 24px, others at 29px). This is visually jarring when viewed side by side.

**Rule**: After all translations are applied, audit font sizes across all artboards:
```js
// Collect all H1 title font sizes
const titleFontSizes = selectedFrames.map(ab => getTitleLayer(ab)?.style.fontSize);
// If they differ, normalize to the most common / largest value
const targetFs = Math.max(...titleFontSizes);
```

**Resolution priority** when a title can't fit at the target font size:
1. **Prefer 2-line layout** at target font size (move group to `y: 18`) — especially valid when the source-language original was also 2-line
2. **Use `fixedWidth = false`** to allow the title to expand beyond its original container (if artboard has space)
3. Only reduce font size as a **last resort**, and only if the minimum readable size would still be ≥ `minFontSize`

---

## Rule 2c — Visual Container vs Logical Parent (视觉容器校验)

The immediate parent `frame` is not always the visual boundary. In decorative components (speech bubbles, cards, overlays), the actual visible area is defined by a **sibling ShapePath**, not the parent group dimensions.

**Example**: A text layer inside a chat bubble Group where:
- `parent.frame.width = 159px` (group, logical boundary)
- sibling `ShapePath "矩形" x=7, width=145` → visual right edge = 152px

The text right edge must not exceed the **visual container** (ShapePath right edge), not the logical parent:

```js
// Find the visual bounding shape (sibling or first ShapePath in parent)
const visualShape = layer.parent.layers.find(
  l => l.type === 'ShapePath' && l.frame.width > layer.parent.frame.width * 0.7
);
const maxRight = visualShape
  ? visualShape.frame.x + visualShape.frame.width
  : layer.parent.frame.width;

const overflow = (layer.frame.x + layer.frame.width) - maxRight;
if (overflow > 0) { /* fix needed */ }
```

**When this applies**: Any text inside a styled container with a background shape — banners, tooltips, notification bars, comment bubbles, tag labels.

---

## Rule 3 — Symbol Master Sync (Symbol 溯源同步)

Before modifying any text layer, check if it's a Symbol instance.

```js
// Check: is this text inside a Symbol?
if (layer.type === 'SymbolInstance' || layer.parent?.type === 'SymbolInstance') {
  // Must update via sketch.find('Text', doc) to reach Symbol Master
  // Exact-match for regular artboard text; keyword-match for Symbol Masters
  // (Symbol Masters store real newlines charCode 10, not '\n' literals)
}
```

**Rule**: Artboard-scoped text → exact full-text match. Symbol Master text → `text.includes(keyword)` match.

Always scope changes to `doc.selectedLayers.layers` — never `sketch.find('Text', doc)` globally unless intentionally targeting Symbol Masters.

**Duplicate artboard names**: `sketch.find('[name="en/彩色/7"]', doc)` may return multiple results when the document has copies of artboards (e.g. different language pages). Always identify the target by additional criteria:

```js
const results = sketch.find('[name="en/彩色/7"]', doc);
const ab = results.find(r => r.parent.name === 'target-page-name');
// or by known coordinates:
const ab = results.find(r => Math.round(r.frame.y) === -2208);
```

---

## Rule 4 — Style Mutation Safety (样式安全赋值) ⚠️

**NEVER** reassign the entire `style` object via spread. It converts the rich SketchAPI Style object to a plain JS object, silently discarding `textColor`, shared style links, color variables, and other non-enumerable properties.

```js
// ❌ WRONG — destroys textColor and other style properties
titleLayer.style = { ...titleLayer.style, fontSize: 24 };

// ✅ CORRECT — mutate individual properties directly
titleLayer.style.fontSize = 24;
titleLayer.style.textColor = '#000000ff';
titleLayer.fixedWidth = false;
```

This rule applies to every style property: `fontSize`, `textColor`, `lineHeight`, `alignment`, etc. Always set them one by one on the existing style object.

---

## Rule 5 — Validation Loop (验证闭环)

After all replacements and repositioning, run this checklist:

```js
function validate(layer, abName) {
  if (layer.type === 'Text') {
    const f = layer.frame, pf = layer.parent?.frame;
    if (!pf) return;
    if (f.x < -2 || f.x + f.width > pf.width + 2 ||
        f.y + f.height > pf.height + 2) {
      console.log(`OVERFLOW: ${abName} > ${layer.name}`);
    }
    if (!layer.fixedWidth && layer.frame.width > pf.width) {
      console.log(`AUTO-WIDTH EXPAND: ${abName} > ${layer.name}`);
    }
  }
  if (layer.layers) layer.layers.forEach(l => validate(l, abName));
}
```

Check:
- [ ] No text overflows parent container
- [ ] `fixedWidth` layers haven't silently expanded
- [ ] No two text layers overlap (y-ranges intersect within same parent)
- [ ] All artboards render identically in preview (`mcp__sketch__get_selection_as_image`)
- [ ] Text colors unchanged from original (compare against untranslated reference artboard if available)
- [ ] No unnecessary `\n` in translated text — Chinese/short translations often don't need line breaks that existed in longer English originals

---

## Language-Specific Notes

### Portuguese (Brazilian)
- Verb phrases expand significantly: "Record" → "Gravar" (+0%), "Share to YouTube" → "Compartilhar no YouTube" (+35%)
- Accented characters (ã, ç, é) don't affect width materially; safe to ignore
- Preferred line-break points: before prepositions (em, no, para, com)

### Indonesian (Bahasa Indonesia)
- Prefixed verbs are long but single-word: "memanfaatkan", "menggunakan" — reduce font before breaking
- Compound nouns are written as one word (unlike German but similar expansion effect)
- Preferred line-break points: after root word boundary, before modifier
- Number formatting: use period as thousands separator (1.000), comma as decimal

### Spanish (LATAM)
- Articles + noun clusters expand predictably; safe to break before article
- Inverted punctuation (¡ ¿) adds 1 character; account for in width checks
- Preferred line-break: before conjunction (y, o, pero, para)

### Russian (RU)
- Expansion vs English: +20–35% for titles; small UI labels can be +50% (e.g. "直播" 2 chars → "Прямой эфир" 11 chars)
- Cyrillic characters are significantly wider than CJK at the same pt size — always re-measure after CJK→RU replacement
- Do **not** translate UI chrome labels that are already in English (e.g. "Live", brand names, social platform names) — keep them as-is
- Preferred line-break points: before prepositions (в, на, с, и, для, без, по), before adjectives
- Verb-heavy translations can be abbreviated: "Подпишись на меня!" fits; "Подпишитесь на наш канал!" is likely too long
- Watch for small-container elements (bottom nav tabs, tooltips, comment bubbles): Russian labels routinely need abbreviation — use dot notation ("Инстр.", "Скор.", "Монт.") for nav labels under ~30px width
- Subtitle font floor: 14px; Title (H1) font floor: 22px — never go below these for Russian

---

## Execution Checklist

When translating a set of artboards:

1. **Scope**: Iterate only `doc.selectedLayers.layers` — never the full document
2. **Translate**: Apply exact full-text map (never `text.includes` for artboard text)
3. **Check newlines**: If translation map contains `\n`, verify the translated string actually needs it — remove if the translated text is shorter and fits single-line
4. **Measure**: Check each Text layer's new `frame.height` and `frame.width`
5. **Fix overflow**: `fixedWidth=false` first → accept 2-line layout → scale font (ratio method) → line-break if still overflowing
6. **Visual container check**: For text inside decorative components, verify right edge against sibling ShapePath bounds, not just parent group width
7. **Reposition**: Recalculate group y and subtitle y dynamically, using `y < 150` guard to avoid inner UI groups
8. **Cross-frame audit**: After all frames are processed, compare font sizes of same-role layers across artboards — if they differ, normalize to the largest valid size using 2-line layout
9. **Style mutations**: Only direct property assignment (`layer.style.fontSize = x`), never spread-reassign
10. **Sync Symbols**: If any text unchanged, check if it's in a Symbol Master
11. **Validate**: Run overflow scan + color check on all modified artboards
12. **Verify visually**: Screenshot via `mcp__sketch__get_selection_as_image`

Reference test cases: [references/test-cases.md](references/test-cases.md)
