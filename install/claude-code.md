# Installing Apex Grid Skill for Claude Code

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/apexcharts/apexgrid-skill.git
```

Claude Code will automatically detect `SKILL.md` and load it when working on `apex-grid` code.

## Verification

> Build a sortable, filterable Apex Grid for a `User` table with a custom avatar cell template.

Claude should:
- Register the custom element via `import 'apex-grid/define'` (or `ApexGrid.register()`) at app startup
- Define a `ColumnConfiguration<User>[]` with `sort: true` and `filter: true` per column, plus the right `type`
- Return a Lit `html\`...\`` template from `cellTemplate`, not a string
- Set `.data` and `.columns` as **properties** on the `<apex-grid>` element, not as attributes
