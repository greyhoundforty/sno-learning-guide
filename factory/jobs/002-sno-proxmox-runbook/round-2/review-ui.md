VERDICT: PASS

# UI/presentation review: 002-sno-proxmox-runbook, round 2

Scope: `docs/sno-proxmox.md` and `.markdownlint.yaml` on `factory/002-sno-proxmox-runbook` (diff vs `main`: 4 files, 597 insertions). This job has no visual UI, so I judged document structure and markdown rendering. I read the whole file and ran `npx markdownlint-cli2 docs/sno-proxmox.md`, which reported 0 issues.

## Round-1 note follow-up

1. **Run-on blockquote (old lines 55–58):** fixed. Lines 58–62 now separate "SeaBIOS alternative" and "QEMU Guest Agent" with a `>` line, so they render as two paragraphs. The blockquote at lines 225–232 ("Tip" and "After installation") uses the same pattern.
2. **Heading numbering gap (Prerequisites, Reference Links):** unchanged. This was acceptable in round 1 and still is.
3. **SNO docs link pointing at 4.13:** not a presentation issue. The text now says "4.13 docs; update link to 4.22 once published", so the mismatch is disclosed.

## Checks

- **Heading hierarchy:** pass. There is one H1. The H2s are Prerequisites, 1–9 and Reference Links. H3s (2.1–2.3, 3.1–3.2, 4.1–4.7, 5.0–5.3) nest under their H2 without skipping a level. The new **5.0** "Wait for cluster operators" is a proper H3 and sorts ahead of 5.1, so the outline reads in order. The "5.0" numbering is unusual but intentional, and it avoids renumbering 5.1–5.3 and the cross-references.
- **Tables:** pass. All five tables have a header row, a separator row and a consistent column count. Cell content is escaped correctly: backticked placeholders such as `<node-ip-address>` and the `*.apps` wildcard are inside code spans. The long troubleshooting cells (lines 364–368) are wide in raw text but render normally in a GitHub-style renderer.
- **Code fences:** pass. All 16 fence markers pair up into 8 blocks, and each opener has a language tag (`text`, `ini`, `bash`). The new 5.0 block (`oc get clusteroperators`) is tagged `bash`. The fences nested in the section 5.2 and section 6 list items are indented to the list-content column and separated by blank lines, so they stay attached to their steps.
- **Blockquotes:** pass. Every blockquote starts with a bold label or plain prose, and the multi-note ones are split by `>` lines.
- **Lists and checklist:** pass. Numbered steps, the nested bullets in 4.2 step 3 and the section 9 task checklist are consistently indented. The checklist continuation lines are indented 2 spaces.
- **Placeholders:** consistently backticked (`<cluster-name>`, `<base-domain>`, `<node-ip-address>`, `<gateway-ip>`, `<dns-ip>`, `<path-to-kubeconfig>`), so nothing is swallowed as HTML.
- **Line length and whitespace:** lines over 80 characters are limited to table rows, one code-block comment and URL-only or link lines with no whitespace past column 80. MD013 passes, with tables and code blocks exempted in `.markdownlint.yaml`. No trailing whitespace was found.
- **Links:** inline and reference-style links are well formed, and `../mise.toml` is correct from `docs/`.
- **Terminology and bold usage:** consistent. UI elements are bold and commands, paths and values are in code spans.

## Minor, non-blocking notes

1. **Flow inside new 5.0:** the section says to run `oc get clusteroperators` "before downloading credentials", but `oc` needs `KUBECONFIG`, which is only obtained and exported in 5.2. A reader following the document top to bottom would hit an auth error at 5.0. This is a content and ordering issue for the content reviewers, not a rendering issue. A simple fix is to move 5.0 after 5.2, or to say "after completing 5.2". If 5.0 is moved, the heading numbers will need adjusting.
2. **Redundant verification:** 5.0 and 5.2 step 4 both run read-only `oc` checks (`oc get clusteroperators`, `oc get clusterversion`). This is harmless, and a reorder per note 1 would make it read naturally.

Nothing here blocks the document. It renders cleanly and passes the linter.
