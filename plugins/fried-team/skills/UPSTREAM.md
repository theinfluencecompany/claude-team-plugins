# External Skill Dependencies

This directory contains skills vendored from external sources for team-wide standardization.

## Superpowers Skills (claude-plugins-official)

**Source**: https://github.com/anthropics/claude-plugins-official (superpowers plugin v4.2.0)
**Vendored**: 2026-02-23
**Skills**:
- `brainstorming` - Explore user intent and requirements before implementation
- `test-driven-development` - TDD workflow for features and bugfixes
- `systematic-debugging` - Root-cause-first debugging process
- `using-git-worktrees` - Isolated feature work with git worktrees
- `writing-plans` - Multi-step task planning
- `executing-plans` - Execute written plans with review checkpoints
- `dispatching-parallel-agents` - Parallel task execution
- `requesting-code-review` - Request review of completed work
- `receiving-code-review` - Handle code review feedback
- `finishing-a-development-branch` - Merge/PR/cleanup decision workflow
- `verification-before-completion` - Verify claims before committing
- `subagent-driven-development` - Execute plans with independent tasks
- `using-superpowers` - Introduction to skills system
- `writing-skills` - Create and edit skills

**To update**: `/plugin update superpowers@claude-plugins-official`, then copy updated skills.

## Vercel React Native Skills

**Source**: https://github.com/vercel-labs/agent-skills
**Skill**: `vercel-react-native-skills`
**Vendored**: 2026-02-23

React Native and Expo best practices covering list performance, animations, navigation, and platform-specific optimizations.

**To update**:
```bash
npx skills add vercel-labs/agent-skills@vercel-react-native-skills -g -y
cp -r ~/.agents/skills/vercel-react-native-skills/ plugins/fried-team/skills/
```

## Team-authored Skills

- `code-audit` - Deep codebase auditing (architecture + implementation)
- `hono` - Hono application development
- `hono-cloudflare` - Hono on Cloudflare Workers
- `hono-rpc` - Hono RPC type-safe APIs
- `convex` - Convex development (umbrella)
- `convex-best-practices` - Convex production guidelines
- `langfuse` - Langfuse LLM observability
- `langfuse-observability` - Langfuse instrumentation
- `workers-best-practices` - Cloudflare Workers patterns
- `wrangler` - Cloudflare Workers CLI
