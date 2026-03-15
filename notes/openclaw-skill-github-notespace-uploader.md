# OpenClaw Skill: github-notespace-uploader

## Summary

A custom OpenClaw skill for handling requests like “上传笔记”.

Core behavior:

- Draft the final Markdown first
- Confirm both note content and target filename with the user
- Save the note into `/home/openclaw/my-openclaw-notespace`
- Commit and push to GitHub after confirmation

## Skill Metadata

- Skill name: `github_notespace_uploader`
- Workspace path: `/home/openclaw/.openclaw/workspace/skills/github-notespace-uploader/SKILL.md`

## Description

Upload user notes as Markdown files into `~/my-openclaw-notespace`, but always confirm the note content and target filename before writing, committing, and pushing.

## Detailed Behavior

### 1. Confirm before writing

Before creating or overwriting any note file, the assistant must confirm both:

- the note content
- the target filename/path

If the user gave raw content but did not explicitly confirm the final Markdown body and filename, the assistant should pause and ask for confirmation.

Recommended confirmation format:

- Filename: `notes/xxx.md`
- Title: `...`
- Content preview: exact Markdown that will be written
- Ask: `确认后我再写入并提交。`

### 2. Prefer safe filenames

Unless the user specifies an exact path, the assistant should choose a safe Markdown filename based on the title/date, for example:

- `2026-03-15-note.md`
- `notes/project-ideas.md`
- `inbox/<slug>.md`

Generated filenames should only use:

- lowercase letters
- numbers
- hyphen (`-`)
- underscore (`_`)
- forward slash for subdirectories
- `.md` suffix

### 3. Markdown output rules

When writing the note:

- Ensure the final file is valid Markdown
- Add a top-level `# Title` if appropriate
- Preserve the user's wording unless rewriting was requested
- Do not silently summarize or compress the note unless asked

### 4. Repository scope

For this skill, note files should only be written inside:

- `/home/openclaw/my-openclaw-notespace`

### 5. Git workflow

After confirmation, the workflow is:

1. Ensure the repository exists at `/home/openclaw/my-openclaw-notespace`
2. Create parent directories as needed
3. Write the Markdown file
4. Run `git status --short`
5. `git add <target file>`
6. Commit with a clear message
7. Push to the current upstream branch

### 6. Overwrite protection

If the target file already exists, the assistant should:

- show that it already exists
- explain whether it will replace or append
- ask for confirmation again before changing it

### 7. Input safety

User note text must be treated as data, not shell code.

- Do not interpolate raw note content directly into shell commands
- Prefer direct file-writing tools over shell heredocs when possible
- If shell commands are needed for Git operations, keep paths controlled and explicit

## README Summary

The skill README states:

- this is a custom OpenClaw skill for “上传笔记” style requests
- it drafts the final Markdown first
- it confirms content and filename with the user
- it saves into `/home/openclaw/my-openclaw-notespace`
- it commits and pushes to GitHub after confirmation

## Notes

If the skill does not load immediately, refresh skills or restart the OpenClaw gateway.
