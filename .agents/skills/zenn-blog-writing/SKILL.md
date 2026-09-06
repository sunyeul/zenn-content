---
name: zenn-blog-writing
description: Use when drafting, revising, structuring, polishing, or preparing a Japanese Zenn technical article, including title/table-of-contents ideation, story framing from a project or paper, figure placement, references, front matter, and casual sharing copy for Slack or social posts.
---

# Zenn Blog Writing

## Overview

Use this skill to turn technical work into a Zenn-ready Japanese article. Choose the structure for the article’s role, and make the reader’s reason to care clear before implementation details. For build and adoption stories, start from the problem and what changed; for tutorials, establish what the reader can try and learn. Keep claims restrained and evidence concrete.

## Core Principle

Center the reader’s task: what they want to do, what the technology now makes possible or easier, and what this article helps them verify. Match that promise to the article’s role. A hands-on introduction earns its value through an accessible starting point and useful comparisons; an experiment post earns it through a question, evidence, and interpretation. Do not require an original finding from a tutorial or pull a planned follow-up experiment into its scope.

## Workflow

1. Identify the article role, reader, and promise.
   - Infer these from the request and current artifacts; ask only when ambiguity materially changes the work.
   - Formulate the reader’s situation, newly possible action, and intended takeaway before choosing sections. Keep this planning concise rather than adding a separate audience section by default.
   - For revisions, establish what must stay unchanged and whether the user wants shorter prose. Preserve boundaries between this article and any planned follow-up.
2. Gather source material.
   - Read current local artifacts before revising claims: notebooks, stored outputs, figures, reports, code, and environment details. Prefer them over older notes; distinguish stored execution evidence from a fresh run and reconcile version differences where supported.
   - Find evidence of the promised change: a working example, before/after comparison, or observed result. Do not infer execution from code or use output placeholders as findings.
   - When code must remain untouched, preserve fenced code and output blocks exactly and compare them before and after editing. Verify existing result numbers and image/link targets separately; change them only within the authorized scope.
   - Browse only when the article relies on current platform rules, external papers, product docs, or public references.
   - For papers or external docs, cite the primary source.
3. Choose the story shape.
   - Default to issue-driven storytelling for tooling, implementation, adoption, and AI-agent experiment posts.
   - Show a concrete before/after early when the value is not obvious: what was hard to inspect, operate, explain, or trust before, and what became easier after.
   - Keep implementation/design lightweight until the reader understands why the change matters.
   - For tutorials and API walkthroughs, give the reader a brief reason to try the example, then move into the runnable sequence. Avoid forcing an invented pain story or an original-research structure onto usage guidance.
4. Draft in Japanese for Zenn.
   - Use `##` as the first visible heading level after front matter.
   - Avoid overly flat outlines. Use `##` for major story beats, `###` for sub-questions or phases inside that beat, and `####` for compact examples, caveats, or before/after details.
   - Put concrete inputs and the purpose of the comparison before array shapes or API details, unless the reader needs a lookup reference.
   - Keep explanations economical: replace jargon with a useful explanation while removing nearby repetition. When readability or brevity is requested, compare prose length excluding code/output blocks; use growth as a review signal, not a fixed reduction target.
   - Give each experiment a readable result and its meaning, followed by the relevant limit. “Look at the graph” and caveats alone do not answer the question. Separate observed result, interpretation, and speculation.
5. Add Zenn front matter when creating an article file.
   - Include `title`, `emoji`, `type: "tech"` or `"idea"`, `topics`, and `published: false` for new drafts; preserve existing publication settings unless asked to change them.
   - Match the title and topics to the final reader promise and actual scope. Keep topics short and lowercase where natural.
6. Add links and references.
   - Use GitHub file links for repo artifacts when the article targets public readers.
   - Prefer commit-pinned links for reproducibility; honor explicit requests for `main` links, including pre-merge drafts, without claiming that the destination is already available.
   - For runnable tutorials, put a notebook or equivalent execution link near the first useful entry point. Derive repository paths from the actual project and verify them.
   - Include primary paper/source links in `参考`.
7. Decide figures sparingly.
   - Prefer 1 concept figure near the motivation/design section and 1 result figure near results.
   - Avoid adding every report chart if a table and one summary figure already communicate the message.
8. Polish for publication and sharing.
   - Revisit title, introduction, and conclusion together: they should promise and answer the same question. Remove duplicated agendas and benefit summaries; keep final results within the comparisons actually performed.
   - Before delivery or publication, use [references/zenn-publication-checklist.md](references/zenn-publication-checklist.md) for the final artifact checks. Report static checks, saved outputs, fresh execution, and rendered inspection according to what was actually verified.
   - **REQUIRED SUB-SKILL:** Use cognitive-rhythm-writing when drafting or revising body prose. Apply it after the article's structure, evidence, and claims are stable; preserve this skill's Zenn, terminology, link, and claim-discipline requirements.
   - Skip the sub-skill when the task covers only an outline, title options, front matter, references, or sharing copy and no article body prose is being drafted or revised.
   - When asked, draft casual Slack/social copy separately from the article.

## Zenn Markdown Notes

Use official Zenn docs for details when needed:

- Markdown guide: https://zenn.dev/zenn/articles/markdown-guide
- Zenn CLI guide: https://zenn.dev/zenn/articles/zenn-cli-guide

Useful reminders:

- Image syntax supports alt text, width hints, and captions on the following line.
- A GitHub file URL on its own line can render as an embed; inline links are better for light references.
- Mermaid is supported, but keep diagrams short and simple.
- Use short bullets for parallel conditions or findings, numbered lists for steps, and tables for real comparisons. Keep causal reasoning and interpretation in connected prose where a list would fragment them.
- Use `:::details` for optional background and `:::message` sparingly for essential caveats. Do not hide prerequisites in collapsed sections.
- If code dominates length, propose collapsing ancillary code or moving it to the notebook rather than compressing necessary explanations. Respect code-preservation and visibility requirements, and keep the runnable order intact.

## Article Design Patterns

### Default: Issue-Driven Technical Post

Use this for most build/tooling/adoption posts, especially when the article could otherwise become "I added X." The main value is solving an operational, explanation, observability, reliability, or decision-making problem.

```markdown
## はじめに
## 困っていたこと
### 具体例: 導入前は何がつらかったか
### 欲しかったもの
## 導入後に何が変わったか
### Before
### After
### 変わらなかったもの
## 実装は最小限にした
### 設計方針
### 実装した範囲
### あえてやらなかったこと
## 検証したこと
### 確認できたこと
### まだ言えないこと
## 分かったこと
## 注意点
## まとめ
```

Make the central issue explicit before naming the tool. Prefer framing like "the evidence exists, but the decision flow is hard to reconstruct" over "added a tool." Include a small before/after table when it clarifies the benefit. Use implementation details to explain tradeoffs, not as the main story.

### Issue-Driven AI Agent / Experiment Post

Use this when writing about AI-agent workflows, skill improvement, evaluations, traces, or harnesses. Show role boundaries and evidence early so the reader can tell what was actually verified.

```markdown
## はじめに
## 何を「改善した気がする」で済ませたくなかったのか
## 導入前の状態
### 見えなかったもの
### 判断しづらかったもの
## 今回欲しかった観測・証拠
### artifact で残したいもの
### trace で見たいもの
## やったこと
### 変更した範囲
### 変更しなかった範囲
## 導入後に確認できたこと
### artifact で確認したこと
### trace で確認したこと
## まだ言えないこと
## まとめ
```

Separate evidence from interpretation. For example, artifacts may prove what remained, while traces may show how a decision flow progressed. Do not let the tool imply stronger claims than the artifacts support.

### Paper-Inspired Project

Use this when the post starts from a paper but the main value is the user's experiment.

```markdown
## はじめに
## 論文のざっくりした話
### この記事で拾う範囲
### 深追いしない範囲
## そこに自分の小さな疑問があった
## 作ったもの
### 最小構成
### 設計上の割り切り
## 実験設計
### 比較条件
### 評価方法
## 結果
### 観測できたこと
### 期待と違ったこと
## 何が効いたのか
## 限界
## まとめ
## 参考
```

Keep the paper summary lightweight. Use the user's work as the spine of the article.

### Usage-Oriented Tooling Post

Use this when the reader mainly needs usage, structure, or operational guidance. For a beginner hands-on, briefly establish the benefit, then organize around runnable examples and their results; omit design or operations sections that do not serve that goal.

```markdown
## はじめに
## 何に困っていたか
## 設計方針
### 優先したこと
### 捨てたこと
## 実装したもの
### 主要コンポーネント
### 最小の使い方
## 実際の使い方
### 基本フロー
### 失敗時の見方
## 運用して分かったこと
## まとめ
```

### Experiment Post

```markdown
## はじめに
## 仮説
## 実験環境
### 対象
### 条件
## 比較条件
### Before
### After
## 結果
### 観測結果
### 代表例
## 解釈
### 何が言えるか
### まだ言えないこと
## 限界
## 次にやりたいこと
```

If a flat `##` outline contains adjacent sections that are really stages of one larger flow, group that flow under one `##` and demote the stages to `###`. Use `####` for small repeated details such as one before/after example, one trace attribute group, one caveat, or one command output summary. Keep optional omake, beyond-MVP, or exploratory sections near the conclusion when they would otherwise interrupt the main argument.

## Experiment Article Checklist

For experiment, paper-implementation, or tool-validation posts, make the body understandable without opening artifact links. Check the question, setup, comparison, observed result, interpretation, and limit at the level the claim requires. For original experiments, include the hypothesis and representative cases when relevant; a hands-on demonstration need not establish novelty or general superiority.

Keep infrastructure, experiment discipline, and individual experiment results in separate sections. When improvement is the point, show what changed in prose, a small table, or a short before/after excerpt before linking to the full artifact.

When an article already has several tables, prefer bullets for light appendix-like notes unless a table adds real comparison value.

## Issue-Driven Storytelling Checklist

Before finalizing most outlines or drafts, check that the article answers:

- Why would the reader care before they know the implementation?
- What was painful, invisible, slow, risky, or hard to explain before?
- What changed after the intervention, in one concrete before/after example?
- What can the chosen tool reveal that was previously only reconstructed manually?
- Is the implementation section serving the issue, rather than becoming the protagonist?
- Are limits and non-goals clear enough to avoid overclaiming?

## Common Failure Modes

- Technology-first opening: "I added X" before explaining why X mattered.
- Implementation as protagonist: architecture, dependencies, and code structure crowd out the problem.
- No before/after: the reader cannot tell what became easier, safer, faster, or more observable.
- Tool overclaiming: the article implies the tool proves more than the verified evidence supports.
- Hidden reader value: useful details exist, but the intro does not say why anyone should care.

## Terminology and Link Discipline

Preserve essential technical terms and exact API identifiers. Lead with a natural Japanese label, introduce the English term once when useful, and reuse the label consistently. Explain unfamiliar terms where the reader needs them; avoid duplicating the same definition in headings, prose, lists, and captions. An analogy may introduce a concept but does not replace its meaning.

For figure text, use the article's body language and reuse the exact terms already chosen in the prose.

Links support explanation; they do not replace it. Summarize the relevant file, commit, paper, or result in the article first. Keep mid-body GitHub file links sparse, and move verification or extra exploration links to `参考` when they would interrupt the reading flow.

## Figure Guidance

Before adding figures, state what each figure should save the reader from mentally reconstructing.

Good figure candidates:

- One concept diagram for the central mechanism.
- One architecture/workflow diagram for role or data boundaries.
- One summary result figure when results are central.

For result figures, summarize the key signal instead of repeating the full result table.

Avoid:

- More than 2-3 figures in a medium-length Zenn article.
- Duplicating a table with several near-identical charts.
- Dense labels that will be unreadable on mobile.

When editing generated figures, prefer adding final labels deterministically with a local image tool or design app rather than relying on generated text.

## Claim Discipline

Use precise language:

- `今回の実験では...` for observed results.
- `示唆しています` for plausible interpretation.
- `まだ言えません` for unsupported causal/general claims.
- Separate predictive result from causal-process evidence.

For experiments with AI agents, explicitly describe context boundaries, allowed artifacts, and role separation if those affect validity.

## Sharing Copy

For Slack or casual internal sharing, avoid a hard-sell tone. Use:

```markdown
久々にブログを書きました〜

[paper/tool/project] を読んで気になった、
「[small question]」
という問いを、[small experiment/tool] を作って試してみました。

[practical design points] みたいな話も書いています。
こういう設計が参考になりますよ〜くらいの内容です。

▼記事はこちら
```

Keep sharing copy shorter than the article intro and more casual than the article body.
