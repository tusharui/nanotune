---
"@nanocollective/nanotune": patch
---

Reject a validation ratio that would leave `train.jsonl` empty.

`splitTrainValidation` computed `trainCount` straight from the ratio and used it
as a `slice` bound, so two inputs drove it to rewrite `train.jsonl` with nothing
in it:

- A ratio of `NaN` makes `Math.max` return `NaN`, so `trainCount` is `NaN` and
  `slice(0, NaN)` is the empty array.
- A ratio of 1 or more makes `trainCount` zero or negative, and `slice` reads a
  negative end as an offset from the end of the array, so the train half comes
  back empty too.

Either way the examples are still on disk, having silently stopped being trained
on, and because `valid.jsonl` now exists `ensureValidationSet` never splits again
so nothing corrects it. The returned counts describe neither: a ratio of 1.2
over ten examples reports `trainCount: -2` and `validCount: 12` for files holding
eight and two.

The ratio is now range-checked before anything is read or written. Ratios of 0
and just under 1 are unaffected and still split as they always did, and for any
ratio in range the split is byte for byte what it was before.

No example is ever dropped or duplicated by this — the two halves always
partition the dataset. The loss was that the train half became empty while the
examples were still being counted as trained on.

Nothing on the CLI passes a user-supplied ratio today; `ensureValidationSet`
always passes 0.1. This is hardening the function ahead of the first caller that
does.
