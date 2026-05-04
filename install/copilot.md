# Installing Apex Grid Skill for GitHub Copilot

## VS Code + Copilot Chat

### Option 1: Add as Workspace Context

1. Clone this repository or download `SKILL.md` into your project
2. In Copilot Chat: `@workspace #file:SKILL.md`
3. Ask your data-grid question

### Option 2: Custom Instructions

1. Open VS Code Settings (Cmd/Ctrl + ,)
2. Search for "Copilot Custom Instructions"
3. Paste the contents of `.cursorrules`

## Verification

Ask Copilot to generate an Apex Grid. It should:
- Import from `apex-grid` (with a hyphen), not `apexgrid`
- Register the custom element exactly once at app startup
- Pass `columns` and `data` via property binding
