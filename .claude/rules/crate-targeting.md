---
paths:
  - "data/refresh.lua"
  - "spec/refresh_spec.lua"
  - "spec/refresh_pipeline_spec.lua"
---
# Crate targeting dimming

## Live evidence and API contract

On November 7, 2026, a local Forever probe printed these values on
`CURRENT_SPELL_CAST_CHANGED`: `SpellIsTargeting()`,
`C_Spell.TargetSpellChecksItemCondition()`, and `ItemButtonUtil.GetItemContext()`.
Activating a crate returned `true true 16`; canceling returned `false false nil`.
Only those flags and the numeric context were retained here.

Forever's UI source (`Blizzard_FrameXMLUtil/ItemUtil.lua`) defines context 16 as
`CheckItemCondition`. Its `GetItemContextMatchResultForItem(location)` asks
`C_Item.DoesItemMatchSpellItemCondition(location)` for that context. Use that
existing resolver; do not infer targets from names or maintain crate contents.

## Refresh behavior

`Refresh:OnEnable` registers one targeting listener when the client exposes
`C_Spell.TargetSpellChecksItemCondition`. While that predicate is true, each
`CURRENT_SPELL_CAST_CHANGED` requests a recalculation, including a direct switch
between crates. The first event after it becomes false also recalculates, clearing
temporary mismatches. Other spell changes do no work.

The serialized refresh queue recalculates the backpack and the open bank from
committed items (`derive`, `categoriesUnchanged`), without a full inventory read,
layout reset, or changes to category configuration. Existing context resolution
and item drawing handle valid targets, invalid targets, and search overlays in
both the standard and Next designs. No new item-button hooks or timers are added.

## Validation boundaries

`spec/refresh_spec.lua` covers requests on start, cancellation, direct switches,
and an open bank. `spec/refresh_pipeline_spec.lua` drives the real scheduler and
item pipeline, checking accepted, rejected, and canceled targeting results.
Existing `spec/frames/item_classic_spec.lua` covers drawing those results and
preserving search dimming independently.

The live probe established the event and context, not individual target verdicts
or the changed UI's appearance. After installation, use `/reload`, activate a
crate with accepted and rejected materials present, and verify their brightness.
Cancel, complete, and switch crates; confirm temporary dimming clears or updates.
Repeat with a search active to verify that search misses remain dimmed.
