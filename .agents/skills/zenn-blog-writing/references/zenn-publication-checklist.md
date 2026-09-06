# Zenn Publication Checklist

Use this checklist before handing off or publishing a Zenn draft.

## Structure

- The opening connects the reader’s situation to what becomes possible and what the article helps them verify.
- The structure fits its role: a runnable introduction need not offer an original research finding.
- Concrete examples precede implementation detail where useful; changes in dataset or comparison scope are clear.
- The title, introduction, and conclusion promise and answer the same question.

## Zenn Fit

- Front matter exists for article files: `title`, `emoji`, `type`, `topics`, `published`.
- Visible headings start at `##`.
- Code blocks have language names and filenames when useful, without altering protected blocks for presentation.
- Lists, tables, messages, and details serve distinct reading needs; required steps remain visible.
- Images have alt text and captions when they carry meaning.
- Figures are readable on mobile.
- Local-only paths are replaced with public links or explained as repo paths.

## Evidence and Links

- External papers/docs link to primary sources.
- Repo references use GitHub links when public readers need them.
- Use commit-pinned links for archival precision unless the user asks for `main` links.
- Results state a supported observation and its meaning, not only a request to inspect a graph or a caveat.
- Current notebook outputs and environment details support the text; saved evidence is distinguished from fresh execution.
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
- Check that references match links used in the body.
- Check that article and sharing copy have different tones.
- Keep new drafts at `published: false`; preserve existing publication settings unless the user asks to change them.
