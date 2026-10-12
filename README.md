# Agent Skills

Agent skills by [Seyi Adedokun](https://github.com/oluwaseyi9325), distilled from building
production React, Next.js and Expo apps. Each skill is a folder with a `SKILL.md` that
teaches an AI coding agent to do one job well. They follow the open
[Agent Skills](https://agentskills.io) format, so they work in Claude Code, Codex, Cursor and
other agents that support it.

## Skills

| Skill | What it does |
|---|---|
| [`react-state-management`](skills/react-state-management/SKILL.md) | Decides where each piece of state lives (server cache, client store, form, URL, local or derived) and implements it with the libraries the project already uses: query keys, instant cache updates after writes, optimistic updates with rollback, workspace-scoped keys, persistence and a complete sign-out. |

## Install

With the [skills CLI](https://github.com/vercel-labs/skills), which detects your agents and
installs into each one's skills folder:

```bash
npx skills add oluwaseyi9325/agent-skills --skill react-state-management
```

Add `-g` to install for your user instead of the current project, or `-a claude-code` (or
another agent's name) to choose the agent.

### Manual install

For Claude Code, copy the skill folder into your project's `.claude/skills/`, or into
`~/.claude/skills/` to use it in every project:

```bash
git clone https://github.com/oluwaseyi9325/agent-skills.git /tmp/agent-skills
mkdir -p .claude/skills
cp -r /tmp/agent-skills/skills/react-state-management .claude/skills/
```

For other agents, copy the same folder into that agent's skills directory. The skills CLI
README [lists the path for each agent](https://github.com/vercel-labs/skills#supported-agents).

The agent loads the skill on its own when a task matches: adding data fetching, a store, a
form or filters; writing a mutation or sign-out; or fixing stale data and needless
re-renders. In Claude Code you can also run it directly with `/react-state-management`.

## `react-state-management`

### What it fixes

- Lists and details that show old values after an edit
- "Did it save?": writes that wait seconds on a refetch of everything
- Server data copied into a store and drifting out of sync
- Re-renders from components reading a whole store or Context
- `useEffect` loops that mirror one piece of state into another
- Edit forms that lose what the user typed when data refetches
- Filters lost on refresh or back
- The previous user's data showing after sign-out
- The wrong workspace's data after switching
- Crashes from old persisted data after an app update

### Libraries covered

The skill reads the project's dependencies and follows them. It doesn't add a second
library for a job that already has one.

| Job | Libraries |
|---|---|
| Server data | TanStack Query, SWR, RTK Query |
| Client state | Zustand, Redux Toolkit, Jotai, React Context |
| Forms | React Hook Form or Formik, with Zod, Yup or Valibot |
| URL state | Next.js, React Router, Expo Router, nuqs |
| Persistence | `localStorage`, MMKV, SecureStore, redux-persist |

Code examples are written for TanStack Query 5, SWR 2, Redux Toolkit 2, Zustand 5, Jotai 2,
React Hook Form 7, Formik 2, Zod 4, Yup 1, Valibot 1 and nuqs 2. It works with React
(Next.js, Vite, React Router) and React Native (Expo).

### How it's organised

`SKILL.md` holds the workflow, where-does-it-live table, rules, a symptoms table and a
final checklist. Library details live in `references/`, and the agent opens only the files
for the libraries your project uses:

```
skills/react-state-management/
├── SKILL.md
└── references/
    ├── server-tanstack-query.md
    ├── server-swr.md
    ├── server-rtk-query.md
    ├── client-zustand.md
    ├── client-redux-toolkit.md
    ├── client-jotai.md
    ├── client-context.md
    ├── forms-react-hook-form.md
    ├── forms-formik.md
    ├── url-state.md
    └── persistence-and-sign-out.md
```

## License

[MIT](LICENSE) © Seyi Adedokun
