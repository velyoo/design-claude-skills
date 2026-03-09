# ax-i18n-pro — Test Cases

Three test cases validating Portuguese (BR) and Indonesian (ID) layout behavior.

---

## Test Case 1 — Portuguese: Title Overflow & Font Scale-Down

**Scenario**: Artboard `pt/彩色/5` has a title "Record with Camera" at `fontSize: 28`, fixed-width container `w: 244px`.

**Input**:
```
Layer: "title"
  text: "Record with Camera"
  fontSize: 28
  fixedWidth: true
  frame: { x: 0, y: 0, width: 244, height: 36 }
  parent.frame.width: 244
```

**Translation**: "Gravar com Câmera" (EN +5%, borderline OK)

**Expected behavior**:
1. Set `layer.text = "Gravar com Câmera"`
2. Measure new `frame.width` — should be ~252px (overflow by ~3%)
3. **Rule 1 triggers**: Reduce `fontSize` by 1pt → 27px
4. Re-measure: width ≈ 241px (fits within 244px) ✅
5. Group stays at `y: 36` (single-line title) ✅

**Pass criteria**:
- No overflow after adjustment
- `fontSize` = 27 (reduced by exactly 1pt)
- Group y unchanged at 36
- Subtitle y unchanged (no height increase)

**Failure modes to catch**:
- ❌ Applying line break before trying font reduction
- ❌ Leaving fontSize at 28 and allowing overflow
- ❌ Moving group y to 18 (unnecessary for single-line result)

---

## Test Case 2 — Portuguese: Two-Line Wrap & Group Repositioning

**Scenario**: Artboard `pt/彩色/3` has title "High Quality, Clear Sound" → translates to "Alta Qualidade, Som Cristalino" (+28%). Container `w: 244px`, `fontSize: 26`.

**Input**:
```
Layer: "title"
  text: "High Quality, Clear Sound"
  fontSize: 26
  fixedWidth: true
  frame: { x: 0, y: 0, width: 244, height: 36 }

Layer: "subtitle"
  text: "Powerful features for all your needs"
  frame: { x: 0, y: 48, width: 244, height: 34 }

Group "text_group"
  frame: { x: 28, y: 36, width: 244, height: 66 }
```

**Translation**: "Alta Qualidade, Som Cristalino"

**Expected behavior**:
1. Set title text → width becomes ~312px (overflow +28%)
2. **Rule 1**: Reduce to 24px → still ~290px (+19% overflow)
3. **Rule 1**: Reduce to 22px → still ~265px (+8% overflow)
4. **Rule 1, step 2**: At 22px still overflows → insert `\n` after comma → "Alta Qualidade,\nSom Cristalino"
5. Title now wraps: height becomes ~58px
6. **Rule 2 triggers**: group moves to `y: 18`, `height: 101`
7. Subtitle y = `title.frame.y + title.frame.height + 12` = `0 + 58 + 12` = **70**
8. Validate: subtitle bottom = 70 + 34 = 104 < 101... recalculate
   - group.height = title.height + 12 + subtitle.height = 58 + 12 + 34 = **104** → set group.height = 104

**Pass criteria**:
- `fontSize` = 22 (reduced from 26)
- Title text contains `\n` at semantic pause (after comma)
- Group `y` = 18
- Subtitle `y` = title height + 12 (dynamic, not hardcoded 48)
- No overlap between title and subtitle

**Failure modes to catch**:
- ❌ Subtitle stays at hardcoded y:48 despite title height change
- ❌ Line break inserted before font reduction attempts
- ❌ Group stays at y:36 after wrapping to two lines
- ❌ Line break at non-semantic position ("Alta Qual\nidade")

---

## Test Case 3 — Indonesian: Prefixed Verb Single-Word Expansion

**Scenario**: Artboard `id/彩色/7` title "Share to YouTube" → "Bagikan ke YouTube". Container `w: 220px`, `fontSize: 28`. Subtitle "Share to your followers" → "Bagikan ke pengikut Anda".

**Input**:
```
Layer: "title"
  text: "Share to YouTube"
  fontSize: 28
  fixedWidth: true
  frame: { x: 0, y: 0, width: 220, height: 36 }

Layer: "subtitle"
  text: "Share to your followers"
  fontSize: 16
  fixedWidth: true
  frame: { x: 0, y: 48, width: 220, height: 20 }
```

**Translations**:
- Title: "Bagikan ke YouTube" (similar length to EN, should fit)
- Subtitle: "Bagikan ke pengikut Anda" (+18% over EN)

**Expected behavior — Title**:
1. "Bagikan ke YouTube" → width ≈ 214px ✅ fits in 220px
2. No font reduction needed
3. Single-line → group stays at y:36

**Expected behavior — Subtitle**:
1. "Bagikan ke pengikut Anda" at 16px → width ≈ 263px (overflow +20%)
2. **Rule 1**: Reduce fontSize to 14px → width ≈ 230px (still +5% overflow)
3. **Rule 1**: Reduce fontSize to 13px → width ≈ 222px (still +1%)
4. **Rule 1**: Reduce fontSize to 12px → width ≈ 211px ✅ fits
5. Check: 12px ≥ 12px minimum for subtitle → ✅ acceptable

**Pass criteria**:
- Title: no change (fits at 28px)
- Subtitle `fontSize` = 12 (reduced from 16, at floor limit)
- No line break added to subtitle (font reduction sufficient)
- Group y stays at 36 (single-line title unchanged)
- Subtitle y stays at 48 (title height unchanged)

**Edge case to verify**: If `minFontSize` for subtitle is set to 13px, then at 13px the subtitle (222px) is 1% over — the skill should insert `\n` before "Anda" (last word, semantically separable): "Bagikan ke pengikut\nAnda", and recheck the two-line title group spec.

**Failure modes to catch**:
- ❌ Treating "Bagikan" as a compound and breaking it mid-word
- ❌ Reducing title font (it fits — no change needed)
- ❌ Stopping font reduction at 14px and calling it "close enough" (+5%)
- ❌ Setting subtitle font below minimum floor (< 12px for body text)
