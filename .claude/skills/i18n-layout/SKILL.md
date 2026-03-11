---
name: i18n-layout
description: This skill should be used when the user asks to translate UI text layers in Sketch, apply multilingual layout rules, handle overflow after translation, fix text overlapping after localization, adapt spacing for Portuguese/Spanish/Indonesian/German/Russian or other languages, or mentions "i18n", "多语言排版", "翻译后溢出", "字体缩小", "副标题重叠", "俄语", "俄文". Also activates when working with Sketch artboards targeting specific regional markets (South America, Southeast Asia, Europe, Russia/CIS).
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

## Frame Type: Screenshot Artboards (截图素材)

When the target frames are **App Store / marketing screenshot artboards** (bitmap output only, not interactive UI):

**You have full editorial freedom:**
- Directly modify Auto Layout settings (gap, padding, alignment) on any group or frame
- Delete unnecessary layers or decorative elements that cause layout issues after translation
- Reposition elements freely — do not try to preserve Symbol override structure
- If a text layer inside a Symbol causes alignment issues, **break the Symbol instance** (`layer.detachStylesAndReplaceWithGroupRecursively()` or just modify the layer directly on the artboard) rather than fighting Symbol overrides
- Prefer simple, direct edits over complex override chains

**Detection heuristic**: Frames are screenshot artboards if they contain phone mockup images, decorative elements (emoji, gradients), and no interactive component annotations. When in doubt, ask the user.

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
3. Reduce `fontSize` proportionally only when 2-line layout is unacceptable. **Ratio calculation is an estimate only — always verify with direct measurement:**
   ```js
   // Step A: estimate (never trust this alone)
   const approxFs = Math.max(currentFs * (targetWidth / currentWidth), minFontSize);
   layer.style.fontSize = approxFs;

   // Step B: directly measure the critical line at this font size
   // (use fw=false temp-text trick — see Quirk 5 for the correct sequence)
   const origText = layer.text;
   layer.text = 'critical line content';   // the longest single line
   layer.fixedWidth = false;
   const actualW = layer.frame.width;      // real width at approxFs
   layer.text = origText;

   // Step C: if actualW still > targetWidth, reduce further
   if (actualW > targetWidth) {
     layer.style.fontSize = Math.max(approxFs * (targetWidth / actualW), minFontSize);
   }
   ```
   **Why**: Font rendering is NOT perfectly linear. Ratio estimates can still overflow by 3–10px. Always do Step B before finalizing.
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

Nav label abbreviations for each language: see **Rule 2d**.

---

## Rule 2 — Relative Layout & Anti-collision (相对布局防重叠)

When a title wraps to a second line, its group height increases. **Never** leave subtitle position hardcoded.

**Title-to-content-frame collision check (⚠️ most common multiline bug)**:

When a title wraps to extra lines, its bottom edge may overlap the content frame (phone screen) below it. Always verify:

```js
// contentFrameY = the y of the phone/content Group (find by ab.layers, type=Group, isFrame)
const titleBottom = title.frame.y + title.frame.height;
if (titleBottom > contentFrameY) {
  // collision — reduce lines (shrink font) or lower contentFrameY
}
```

**Collision resolution order:**
1. Reduce title to fewer lines (smaller font or wider container) so `titleBottom ≤ contentFrameY - 8`
2. If font is already at `minFontSize`, accept the overlap only if the content frame has an opaque background that clips the title

**Subtitle anti-collision** (when title and subtitle are in a group):

```js
// Generic — do NOT use hardcoded y/h values; always derive from measured content
const subtitleY = title.frame.y + title.frame.height + GAP;  // GAP ≈ 12px
group.frame.y = TOP_MARGIN;  // TOP_MARGIN depends on the specific artboard layout
group.frame.height = title.frame.height + GAP + subtitle.frame.height;
subtitle.frame.y = subtitleY;
```

**Critical**: Adjust the *group* position, not individual text layers within the group. Compressing inter-element gaps destroys visual rhythm.

**Title group identification**: When iterating `ab.layers` to find the title group, **only consider direct children with `y < contentFrameY`**. Artboards containing phone screen mockups often have inner UI groups that also contain large-font text — these must not be mistaken for the artboard title group:

```js
// Find content frame start (phone screen Group)
let contentFrameY = 999;
ab.layers.forEach(l => {
  if ((l.type === 'Group' || l.isFrame) && l.frame.y > 100 && l.frame.y < contentFrameY) {
    contentFrameY = l.frame.y;  // topmost content container
  }
});

// Find title group: direct children above content frame with large text
ab.layers.forEach(l => {
  if ((l.type === 'Group' || l.isFrame) && l.layers && l.frame.y < contentFrameY) {
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

## Rule 2d — Nav Tab Label Pre-flight Check (导航栏标签预检)

Bottom navigation bars typically divide ~288px across 5 tabs = **~57px per tab**. Any translated label that exceeds this width will visually overlap adjacent tabs — the most visually destructive layout bug.

**Rule**: Before writing the translation map, measure every nav label's expected rendered width. Any label > 57px must be abbreviated in the translation table upfront — do not wait for overflow detection.

**Width estimation at ~11px font** (5 labels in 288px):
- English baseline chars average ~5.5px/char at 11px
- Cyrillic chars average ~6.5px/char at 11px
- Multi-word labels (containing spaces) almost always overflow

**Abbreviation conventions by language:**

| Language | Full label | Width | → Abbreviated | Width |
|----------|-----------|-------|----------------|-------|
| Russian | Инструменты | 70px | Инстр. | ~39px |
| Spanish | Captura de Pantalla | 95px | Captura | 38px |
| Spanish | Herramientas | 66px | Herram. | 39px |
| PT (BR) | Captura de Tela | 75px | Captura | 38px |
| PT (BR) | Ferramentas | 61px | Ferram. | 37px |
| Indonesian | Tangkapan Layar | 82px | Tangkap | ~42px |

**Convention rules:**
1. If the first word alone is a recognizable standalone noun → use first word, no period (`Captura`, `Tangkap`)
2. If truncation cuts mid-word → add period to signal abbreviation (`Herram.`, `Ferram.`, `Инстр.`)
3. Never abbreviate to fewer than 4 chars unless the full word is already ≤4 chars
4. Verify the abbreviated form is unambiguous in context (app nav bar)

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
- Nav tab abbreviations: see **Rule 2d** (`Captura de Tela` → `Captura`, `Ferramentas` → `Ferram.`)

### Indonesian (Bahasa Indonesia)
- Prefixed verbs are long but single-word: "memanfaatkan", "menggunakan" — reduce font before breaking
- Compound nouns are written as one word (unlike German but similar expansion effect)
- Preferred line-break points: after root word boundary, before modifier
- Number formatting: use period as thousands separator (1.000), comma as decimal
- Nav tab abbreviations: see **Rule 2d** (`Tangkapan Layar` → `Tangkap`)

### Spanish (LATAM)
- Articles + noun clusters expand predictably; safe to break before article
- Inverted punctuation (¡ ¿) adds 1 character; account for in width checks
- Preferred line-break: before conjunction (y, o, pero, para)
- Nav tab abbreviations: see **Rule 2d** (`Captura de Pantalla` → `Captura`, `Herramientas` → `Herram.`)

### Russian (RU)
- Expansion vs English: +20–35% for titles; small UI labels can be +50% (e.g. "直播" 2 chars → "Прямой эфир" 11 chars)
- Cyrillic characters are significantly wider than CJK at the same pt size — always re-measure after CJK→RU replacement
- Do **not** translate UI chrome labels that are already in English (e.g. "Live", brand names, social platform names) — keep them as-is
- Preferred line-break points: before prepositions (в, на, с, и, для, без, по), before adjectives
- Verb-heavy translations can be abbreviated: "Подпишись на меня!" fits; "Подпишитесь на наш канал!" is likely too long
- Nav tab abbreviations: see **Rule 2d** (`Инструменты` → `Инстр.`)
- Subtitle font floor: 14px; Title (H1) font floor: 22px — never go below these for Russian

---

## ⚠️ Sketch JS Engine Quirks (已验证的引擎陷阱)

在 Sketch 插件脚本（JavaScriptCore 定制版）中，以下行为与标准 JS **不同**，必须提前规避：

### 陷阱 1 — 嵌套括号访问会触发 SyntaxError

```js
// ❌ 报 SyntaxError: Unexpected token '['
const lk = lkMap[parts[0]]

// ✅ 必须用中间变量
const p0 = parts[0]
const lk = lkMap[p0]
```

**规则**：任何 `a[b[n]]` 形式的嵌套括号访问都不可用，必须拆成两步。

### 陷阱 2 — `id` 是引擎保留字，读取永远返回 undefined

```js
const obj = { id: 'hello', ru: 'world' }
obj.id      // undefined ← 被引擎拦截！
obj['id']   // undefined ← 同样失效
obj.ru      // 'world'   ← 正常

// ✅ 印尼语（Indonesian）的语言键必须用 ind，不能用 id
const lkMap = { 'pt-BR': 'ptBR', 'ru': 'ru', 'id': 'ind', 'es': 'es' }
const TR = { ptBR: '...', ru: '...', ind: '...', es: '...' }
```

**规则**：永远不要在翻译 map 中使用 `id` 作为属性键名，统一改用 `ind`。

### 陷阱 3 — Symbol 实例的内层 Text 是只读的

```js
// ❌ 会抛 Obj-C 异常：-[MOUndefined length]
symbolInstance.layers[0].text = 'new text'

// ✅ 必须通过 overrides 接口修改
symbolInstance.overrides.forEach(o => {
  if (o.property === 'stringValue') o.value = translation
})
```

**规则**：walk 函数遇到 `SymbolInstance` 时，**停止递归进入其子层**，转为操作 `overrides`。否则看似合法的 `layer.text = x` 会在运行时静默崩溃。

```js
// ✅ 安全的 walk 模板
function walk(layer) {
  if (layer.type === 'SymbolInstance') {
    applyOverrides(layer)  // 用 overrides 接口
    return                  // ← 不再递归进入子层
  }
  if (layer.type === 'Text') {
    layer.text = translate(layer.text)
  }
  if (layer.layers) {
    const ch = layer.layers
    for (let i = 0; i < ch.length; i++) walk(ch[i])
  }
}
```

### 陷阱 4 — 设置 undefined 会污染 override 数据

```js
// ❌ 如果 nv 是 undefined，会把字符串 "undefined" 写入 override
o.value = tr[lk]

// ✅ 必须加守卫
const nv = tr[lk]
if (nv) o.value = nv
```

**规则**：每次赋值前必须检查值是否存在，避免 "undefined" 字符串污染数据后难以清除。

### 陷阱 5 — fixedWidth=true 层修改 fontSize/frame.width 后 frame.height 不同步更新

对 `fixedWidth=true` 的文本层，直接修改 `style.fontSize` 或 `frame.width`，**当前脚本内立即读回 `frame.height` 得到的是旧值（缓存）**。视觉渲染是正确的，但 API 回读不可信，导致后续判断（是否 2 行）全部出错。

```js
// ❌ 以下顺序读到的 h 是缓存值，不可信
layer.style.fontSize = 27
layer.frame.width = 346
const h = layer.frame.height  // ← 可能仍是旧的 126，实际渲染已是 84
```

**✅ 唯一可靠的 fixedWidth=true 重排序列：**

```js
// 步骤 1：设置新字号
layer.style.fontSize = newFs

// 步骤 2：切到 fw=false，读一次 width（强制引擎同步渲染）
layer.fixedWidth = false
const autoW = layer.frame.width  // 此处读取会触发同步重排，autoW = 自然单行宽度

// 步骤 3：切回 fw=true（此时引擎锁定为 autoW）
layer.fixedWidth = true

// 步骤 4：设置目标固定宽度（覆盖 autoW）
layer.frame.width = targetWidth

// 步骤 5：现在读 frame.height 才是准确的
const h = layer.frame.height
```

**关键**：步骤 2 的"读 width"动作是触发同步渲染的关键，不能省略。步骤 4 必须在步骤 3 之后设置，否则会被 fw=true 锁定覆盖。

---

## Execution Checklist

When translating a set of artboards:

0. **JS Engine pre-check** (run first, avoid all downstream bugs):
   - Never use `a[b[n]]` nested bracket access → split into two variables
   - Never use `id` as translation map key → use `ind` for Indonesian
   - Never walk into SymbolInstance children → use `overrides` API, then `return`
   - Always guard `if (nv) o.value = nv` before any override assignment
   - `frame.height` on `fixedWidth=true` layers is unreliable after fontSize/width change → use Quirk-5 reflow sequence
   - Font size ratio estimates are approximate → always directly measure the critical line width before finalizing
   - Pre-check nav tab label lengths: any label > 57px at target font size needs abbreviation in the translation table (see Rule 2d for per-language reference table)
1. **Scope**: Iterate only `doc.selectedLayers.layers` — never the full document
2. **Translate**: Apply exact full-text map (never `text.includes` for artboard text)
3. **Check newlines**: If translation map contains `\n`, verify the translated string actually needs it — remove if the translated text is shorter and fits single-line
4. **Force relayout — different sequences for fw=false vs fw=true**:
   - **fw=false layers** (auto-width): toggle to trigger width recalculation
     ```js
     if (!layer.fixedWidth) { layer.fixedWidth = true; layer.fixedWidth = false; }
     ```
   - **fw=true layers** (fixed-width): direct `frame.width` change does NOT update `frame.height` synchronously. Use the 5-step reflow sequence (see Quirk 5):
     ```js
     layer.style.fontSize = newFs;
     layer.fixedWidth = false;
     const autoW = layer.frame.width;  // ← triggers sync render
     layer.fixedWidth = true;
     layer.frame.width = targetWidth;
     // frame.height is now accurate
     ```
5. **Measure**: Check each Text layer's new `frame.height` and `frame.width`. **Never trust frame.height on fw=true layers unless the Quirk-5 reflow sequence was used.**
6. **Fix overflow**: `fixedWidth=false` first → accept 2-line layout → scale font (**measure directly**, not just ratio) → line-break if still overflowing
7. **Title-body collision check**: After any title line-count change, verify `title.frame.y + title.frame.height ≤ contentFrameY - 8`. If violated, reduce font or lines (see Rule 2)
8. **Visual container check**: For text inside decorative components, verify right edge against sibling ShapePath bounds, not just parent group width
9. **Reposition**: Recalculate subtitle y dynamically from `title.frame.y + title.frame.height + GAP`; detect `contentFrameY` from the actual layer tree, not hardcoded values
10. **Cross-frame audit**: After all frames are processed, compare font sizes of same-role layers across artboards — if they differ, normalize to the largest valid size using 2-line layout
11. **Style mutations**: Only direct property assignment (`layer.style.fontSize = x`), never spread-reassign
12. **Sync Symbols**: If any text unchanged, check if it's in a Symbol Master
13. **Validate**: Run overflow scan + color check on all modified artboards
14. **Verify visually**: Screenshot via `mcp__sketch__get_selection_as_image`

Reference test cases: [references/test-cases.md](references/test-cases.md)
