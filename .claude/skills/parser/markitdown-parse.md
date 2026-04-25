---
name: markitdown-parse
description: Convert any supported document (PDF, MD, DOCX, PPTX, HTML, TXT) to a markdown text string. Invoked by the /ghostcheck router to normalise CV and JD inputs before agents read them. Auto-routes to Claude Code's native Read tool for PDF and Markdown inputs (zero install required) or shells out to the MarkItDown CLI for other formats. Halts with a clear install message if MarkItDown is required but missing.
allowed-tools:
  - Read
  - Bash
---

# Parser skill — file to markdown text

When invoked with a file path, return the file contents as a single markdown text string. Route based on the file's extension. Fail closed on any parsing failure: surface a clear error and halt.

## Step 1 — Determine the file extension

Take the file extension from the path and lowercase it. If there is no extension at all, halt with: `"Cannot parse: file at <path> has no extension. Halt parsing."`

## Step 2 — Route based on extension

### Native path (no install needed): `.md` and `.pdf`

For these two formats, use Claude Code's Read tool directly. No external dependency, no install, no shell-out.

- `.md`: Read the file with the Read tool. The contents are already markdown. Return as-is.
- `.pdf`: Read the file with the Read tool. For PDFs over 10 pages, use the Read tool's `pages` parameter to read in chunks (maximum 20 pages per request). Concatenate the chunks into a single markdown string and return.

### MarkItDown path (requires install): `.docx`, `.pptx`, `.html`, `.txt`

For these formats, shell out to the MarkItDown CLI via the Bash tool.

1. **Check that MarkItDown is installed.** Run `which markitdown` via Bash. If the command is not found (non-zero exit code or empty stdout), halt with this message:

   ```
   This document type requires MarkItDown with format-specific extras.
   Install once with: pip install 'markitdown[all]'

   For a single format only:
     pip install 'markitdown[pptx]'   # PowerPoint
     pip install 'markitdown[docx]'   # Word
     pip install 'markitdown[html]'   # HTML

   Note the single-quotes around the package name. macOS's default
   shell (zsh) treats unquoted square brackets as a glob pattern and
   the install fails with "zsh: no matches found". On bash quoting is
   harmless but on zsh it is required. The bare `pip install markitdown`
   (without extras) installs the CLI but cannot read PPTX, DOCX, or HTML.

   Then re-run /ghostcheck audit.

   PDF and Markdown files work without any install — if you can convert
   your file to PDF, you can avoid the MarkItDown dependency entirely.
   ```

2. **If MarkItDown is found, invoke it on the file.** Run `markitdown "<path>"` via Bash. Capture stdout as the markdown text.

3. **Handle MarkItDown errors.** If MarkItDown exits with a non-zero status, halt with the captured stderr in the message: `"MarkItDown failed for <path>: <stderr message>. Halt parsing."` Pay particular attention to the error `MissingDependencyException: ... include the optional dependency [pptx] or [all]` — this means MarkItDown is installed but without the format-specific extras. Surface the install instructions from step 1 to the user.

### Unsupported extensions

If the extension is anything other than the six listed above (`.md`, `.pdf`, `.docx`, `.pptx`, `.html`, `.txt`), halt with: `"Unsupported file type: <ext>. Halt parsing."`

(The router's Step 1 already filters extensions per its own allow-list, so this branch is a defensive backstop. It should rarely fire in normal use.)

## Step 3 — Return the markdown text

The parser's output to its caller is a single markdown string. The caller (typically the `/ghostcheck` router) holds it as `cv_text` or `jd_text` and continues the audit flow.

## Behavioural rules (invariants the parser must respect)

- **The parser does not classify documents** as CV vs JD vs other. That is the router's Step 3 (LLM classification call), a separate concern.
- **The parser does not validate content quality.** Whether the parsed markdown looks like a real CV is the agents' job, not the parser's.
- **The parser does not write to disk.** It returns text to the caller; the caller decides what to do with it.
- **The parser does not silently install MarkItDown.** If it is missing, halt with the install message; never run `pip install` without explicit user consent. The user must run the install command themselves.
- **The parser does not retry on transient failures.** If a Read or MarkItDown call fails, surface the error and halt. The user re-runs the audit; we do not silently re-attempt.
