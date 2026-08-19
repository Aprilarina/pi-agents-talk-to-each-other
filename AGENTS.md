# AGENTS.md

## Workflow rules

- **Never commit and push a new version without testing the extension locally first.** Run `pi -e . --list-models` to verify the extension loads without errors before any `git commit && git push`.
- After pushing, always run `pi update git:github.com/Aprilarina/pi-agents-talk-to-each-other` to update the installed package.
- Verify the installed version matches the pushed version by checking `EXTENSION_VERSION` in the installed file.