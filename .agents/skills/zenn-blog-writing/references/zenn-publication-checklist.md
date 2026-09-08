# Zenn Publication Checklist

Use this checklist before handing off or publishing a Zenn draft.

## Structure

- The opening connects the reader’s situation to what becomes possible and what the article helps them verify.
- The structure fits its role: a runnable introduction need not offer an original research finding.
- Concrete examples precede implementation detail where useful; changes in dataset or comparison scope are clear.
- The title, introduction, and conclusion promise and answer the same question; the ending leaves usable decision criteria, not only cautions.
- Transitions explain why the next comparison is needed and distinguish models, estimation methods, and evaluation metrics when the comparison changes.

## Zenn Fit

- Front matter exists for article files: `title`, `emoji`, `type`, `topics`, `published`.
- Visible headings start at `##`.
- Code blocks have language names and filenames when useful, without altering protected blocks for presentation.
- Lists, tables, messages, and details serve distinct reading needs; required steps remain visible.
- Images have alt text and captions when they carry meaning.
- Figures are readable on mobile. Figure-reading instructions match display order and account for plots hidden in collapsed sections.
- Local-only paths and production-process notes appear only when readers need them. Runnable notebooks use direct execution links; featured previous articles use link cards according to project conventions.
- Editable numeric displays use appropriate precision, with integer counts; rounding does not hide meaningful small values or alter source outputs.

## Evidence and Links

- External papers/docs link to primary sources.
- Repo references use GitHub links when public readers need them.
- Use commit-pinned links for archival precision unless the user asks for `main` links.
- Experiment reports explain supported observations; rerunnable guides explain outcome patterns without fitting their narrative to one run. Neither relies only on graph-reading instructions or cautions.
- Current notebook outputs and environment details support the text. The work report distinguishes saved evidence from fresh execution; reader-facing provenance is included only where relevant.
- If code preservation was requested, before/after blocks match and existing numeric results and link/image targets have been checked.
- Any causal claim names the evidence and caveats.

## Style

- Japanese prose is conversational but not vague.
- Plain-language explanations replace jargon and repetition rather than expanding each paragraph.
- If brevity was requested, compare prose length excluding code/output and examine any growth; account for code separately.
- Prefer concrete examples over abstract claims.
- Keep English technical terms when they are natural for the audience.
- Avoid too many figures, tables, and bullets in a row.

## Final Pass

- Remove repeated agendas, definitions, and benefit summaries while preserving the result interpretation.
- Evaluate the article against its intended role; do not require content reserved for another article.
- Check that references match links used in the body. For notebook adaptations, reconcile the source bibliography, figures, and tables so that requested material and relevant references are not accidentally omitted.
- Check that article and sharing copy have different tones.
- Keep new drafts at `published: false`; preserve existing publication settings unless the user asks to change them.
