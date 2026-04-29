# AGENT-60: Expose Triggers Configuration in Agent JSON Config

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Linear ticket:** https://linear.app/n8n/issue/AGENT-60/feature-expose-triggers-configuration-to-agentjson

**Goal:** Expose agent triggers (schedule + chat integrations) in `AgentJsonConfig` so the builder agent can read and modify them through `write_config` / `patch_config`, the same way it manages `tools` today.

**Architecture:** Add an `integrations` field to `AgentJsonConfig` (a discriminated union of schedule + chat integrations). Storage stays in the existing `agent.integrations` JSON column — the JSON-config view is **composed** from `agent.schema` + `agent.integrations` on read and **decomposed** back to the column on write. A new `syncIntegrations` step diffs old/new integrations and drives the existing `ChatIntegrationService` / `AgentScheduleService` lifecycle methods. A new `list_integration_types` builder tool lets the LLM discover available platforms.

**Tech Stack:** TypeScript, Zod, NestJS-style DI (@n8n/di), TypeORM, Jest. Builder uses `@n8n/agents` SDK for tool definitions.

**Out of scope:**
- Snapshotting integrations into `AgentPublishedVersion` (today they live on the live agent entity; we keep that).
- New chat integration platforms.
- Frontend changes to the agent settings UI (the existing endpoints continue to work; this change is additive at the schema level).
- Telemetry events (will be tracked separately as part of the A/C — keep hooks ready but don't block on it).

**Design decisions confirmed with user:**
1. Field name is `integrations` (not `triggers`); single discriminated-union array at root.
2. Credentials use `{ credentialId, credentialName }` pair (matches node-tool credentials).
3. DB storage stays in `agent.integrations` column; JSON config wires into it via compose/decompose.
4. Schedule keeps the `active` boolean.
5. Add a `list_integration_types` builder tool.

---

## File Structure

**New files:**
- `packages/cli/src/modules/agents/json-config/integration-config.ts` — Zod schemas for `AgentIntegration` discriminated union (cli-side validation).
- `packages/cli/src/modules/agents/json-config/integration-config.test.ts` — Unit tests for the new schemas.
- `packages/cli/src/modules/agents/__tests__/agent-config-composition.test.ts` — Unit tests for compose/decompose helpers.
- `packages/cli/src/modules/agents/__tests__/integration-sync.test.ts` — Unit tests for the diff/sync logic.
- `packages/cli/src/modules/agents/builder/__tests__/list-integration-types.tool.test.ts` — Unit tests for the new builder tool.

**Modified files:**
- `packages/@n8n/api-types/src/agents.ts` — Add `credentialName` to `AgentCredentialIntegration`; add `integrations` to `AgentJsonConfig`.
- `packages/cli/src/modules/agents/json-config/agent-json-config.ts` — Add `integrations` to `AgentJsonConfigSchema`.
- `packages/cli/src/modules/agents/agents.service.ts` — Compose on read, decompose + sync on write; add `syncIntegrations`.
- `packages/cli/src/modules/agents/integrations/chat-integration.service.ts` — Populate `credentialName` when persisting; expose helpers needed by sync.
- `packages/cli/src/modules/agents/integrations/agent-schedule.service.ts` — Idempotent `applyConfig` helper for sync.
- `packages/cli/src/modules/agents/builder/agents-builder-tools.service.ts` — Register `list_integration_types` tool.
- `packages/cli/src/modules/agents/builder/agents-builder-prompts.ts` — Add `INTEGRATIONS_SECTION` and reference it from the workflow section.
- `packages/cli/src/modules/agents/__tests__/from-json-config.test.ts` — Cover the new field round-trips correctly.
- `packages/cli/src/modules/agents/__tests__/agents-builder-prompts.test.ts` — Cover the new prompt section.

---

## Task 1: Extend public types in `@n8n/api-types`

**Files:**
- Modify: `packages/@n8n/api-types/src/agents.ts`

The shared types are the source of truth between FE and BE. Add `credentialName` to credential-based integrations and add `integrations` to `AgentJsonConfig`. No tests live in this package for these types — typecheck is the gate.

- [ ] **Step 1: Read the current shape**

Run: `cat packages/@n8n/api-types/src/agents.ts | head -120`
Confirm `AgentCredentialIntegration`, `AgentScheduleIntegration`, `AgentIntegration`, and `AgentJsonConfig` interfaces look as expected from the architecture report.

- [ ] **Step 2: Add `credentialName` to `AgentCredentialIntegration`**

Edit `packages/@n8n/api-types/src/agents.ts`:

```typescript
export interface AgentCredentialIntegration {
	type: string;
	credentialId: string;
	credentialName: string;
}
```

- [ ] **Step 3: Add `integrations` to `AgentJsonConfig`**

Inside `AgentJsonConfig`, after `providerTools`, add:

```typescript
	/**
	 * Triggers (scheduled execution + chat integrations) attached to this agent.
	 * Mirrors the contents of `agent.integrations` storage column so the builder
	 * can read and modify triggers through the same JSON config flow as tools.
	 */
	integrations?: AgentIntegration[];
```

- [ ] **Step 4: Run typecheck on api-types**

Run: `pnpm --filter=@n8n/api-types typecheck`
Expected: PASS.

- [ ] **Step 5: Run typecheck on cli (will surface call sites that need updates later)**

Run: `pnpm --filter=n8n typecheck 2>&1 | tail -40`
Expected: errors in `chat-integration.service.ts` and tests where `AgentCredentialIntegration` is constructed without `credentialName`. **These are fixed in later tasks** — this is just a survey.

Capture the failing files into a scratch note for follow-up. Do **not** fix them in this task.

- [ ] **Step 6: Commit**

```bash
git add packages/@n8n/api-types/src/agents.ts
git commit -m "feat(api-types): add integrations field to AgentJsonConfig"
```

---

## Task 2: Add Zod validation for integrations in cli

**Files:**
- Create: `packages/cli/src/modules/agents/json-config/integration-config.ts`
- Create: `packages/cli/src/modules/agents/json-config/integration-config.test.ts`
- Modify: `packages/cli/src/modules/agents/json-config/agent-json-config.ts`

This is the runtime validation gate. It must mirror the api-types interfaces and reject malformed input from the builder.

- [ ] **Step 1: Write failing test**

Create `packages/cli/src/modules/agents/json-config/integration-config.test.ts`:

```typescript
import { AgentIntegrationSchema } from './integration-config';

describe('AgentIntegrationSchema', () => {
	it('accepts a schedule integration', () => {
		const result = AgentIntegrationSchema.safeParse({
			type: 'schedule',
			active: true,
			cronExpression: '0 9 * * *',
			wakeUpPrompt: 'Daily standup ping',
		});
		expect(result.success).toBe(true);
	});

	it('accepts a chat integration with credential name', () => {
		const result = AgentIntegrationSchema.safeParse({
			type: 'slack',
			credentialId: 'cred-123',
			credentialName: 'Acme Slack',
		});
		expect(result.success).toBe(true);
	});

	it('rejects a chat integration missing credentialName', () => {
		const result = AgentIntegrationSchema.safeParse({
			type: 'slack',
			credentialId: 'cred-123',
		});
		expect(result.success).toBe(false);
	});

	it('rejects a schedule integration missing cronExpression', () => {
		const result = AgentIntegrationSchema.safeParse({
			type: 'schedule',
			active: true,
			wakeUpPrompt: 'hello',
		});
		expect(result.success).toBe(false);
	});

	it('rejects a chat integration with type "schedule" (reserved)', () => {
		const result = AgentIntegrationSchema.safeParse({
			type: 'schedule',
			credentialId: 'cred-123',
			credentialName: 'Acme',
		});
		// Schedule branch requires cronExpression — should fail
		expect(result.success).toBe(false);
	});
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pushd packages/cli && pnpm test src/modules/agents/json-config/integration-config.test.ts 2>&1 | tail -20 && popd`
Expected: FAIL with "Cannot find module './integration-config'".

- [ ] **Step 3: Implement the schemas**

Create `packages/cli/src/modules/agents/json-config/integration-config.ts`:

```typescript
import { z } from 'zod';

/**
 * Schedule trigger: fires the agent on a cron schedule with a fixed wake-up prompt.
 * Activation requires the agent to have a published version (enforced by AgentScheduleService).
 */
export const AgentScheduleIntegrationSchema = z
	.object({
		type: z.literal('schedule'),
		active: z.boolean(),
		cronExpression: z.string().min(1, 'cronExpression is required'),
		wakeUpPrompt: z.string().min(1, 'wakeUpPrompt is required'),
	})
	.strict();

/**
 * Chat-platform trigger (Slack, Telegram, Linear, ...).
 * `credentialName` is the user-visible name resolved at write time so the builder LLM
 * never has to round-trip through credential lookup to reason about the config.
 */
export const AgentCredentialIntegrationSchema = z
	.object({
		type: z.string().min(1).refine((value) => value !== 'schedule', {
			message: 'Type "schedule" is reserved for the schedule trigger',
		}),
		credentialId: z.string().min(1),
		credentialName: z.string().min(1),
	})
	.strict();

export const AgentIntegrationSchema = z.discriminatedUnion('type', [
	AgentScheduleIntegrationSchema,
	// `discriminatedUnion` requires literal types on every branch, so we cannot reuse
	// the open-ended `AgentCredentialIntegrationSchema`. Instead we use `union` underneath
	// after the schedule branch is matched first.
	z
		.object({
			type: z.string().refine((value) => value !== 'schedule'),
			credentialId: z.string().min(1),
			credentialName: z.string().min(1),
		})
		.strict(),
]);
```

> **Note on the union shape:** Zod's `discriminatedUnion` requires every branch to use a literal discriminator. Schedule has the literal `'schedule'`; chat integrations use a string with the schedule value rejected. We rely on the schedule branch being checked first so chat integrations never accidentally match it.

- [ ] **Step 4: Run test to verify it passes**

Run: `pushd packages/cli && pnpm test src/modules/agents/json-config/integration-config.test.ts 2>&1 | tail -15 && popd`
Expected: PASS, 5 passing tests.

- [ ] **Step 5: Wire into `AgentJsonConfigSchema`**

Edit `packages/cli/src/modules/agents/json-config/agent-json-config.ts`. Find the root `AgentJsonConfigSchema` (around line 100) and add the `integrations` field next to `providerTools`:

```typescript
import { AgentIntegrationSchema } from './integration-config';

// ...inside AgentJsonConfigSchema definition...
		providerTools: z
			.record(z.string(), z.record(z.string(), z.unknown()))
			.optional(),
		integrations: z.array(AgentIntegrationSchema).optional(),
```

- [ ] **Step 6: Add a round-trip test in `from-json-config.test.ts`**

Append to `packages/cli/src/modules/agents/__tests__/from-json-config.test.ts`:

```typescript
describe('integrations field', () => {
	it('parses through to the runtime config without error', async () => {
		const config = {
			name: 'Sample',
			model: 'anthropic/claude-sonnet-4-5',
			instructions: 'You are a helpful assistant.',
			integrations: [
				{ type: 'schedule', active: false, cronExpression: '0 0 * * *', wakeUpPrompt: 'tick' },
				{ type: 'slack', credentialId: 'cred-1', credentialName: 'Acme Slack' },
			],
		};
		// buildFromJson currently ignores `integrations` (handled at service layer);
		// the assertion is that validation passes and the rest of the config builds.
		const built = await buildFromJson(config, /* toolDescriptors */ {}, makeStubDeps());
		expect(built.name).toBe('Sample');
	});
});
```

(`makeStubDeps()` already exists in that file; reuse it.)

- [ ] **Step 7: Run all json-config tests**

Run: `pushd packages/cli && pnpm test src/modules/agents/json-config 2>&1 | tail -20 && popd`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add packages/cli/src/modules/agents/json-config/integration-config.ts \
        packages/cli/src/modules/agents/json-config/integration-config.test.ts \
        packages/cli/src/modules/agents/json-config/agent-json-config.ts \
        packages/cli/src/modules/agents/__tests__/from-json-config.test.ts
git commit -m "feat(agents): validate integrations in AgentJsonConfigSchema"
```

---

## Task 3: Backfill `credentialName` in existing chat integration writes

**Files:**
- Modify: `packages/cli/src/modules/agents/integrations/chat-integration.service.ts`
- Modify: `packages/cli/src/modules/agents/agents.controller.ts`
- Modify: `packages/cli/src/modules/agents/integrations/__tests__/agent-chat-bridge.test.ts` (or relevant existing test) — assertion for new field
- Modify: `packages/cli/src/modules/agents/__tests__/agents-controller.test.ts` (if exists)

The controller currently appends `{ type, credentialId }` to `agent.integrations`. With the new schema requiring `credentialName`, we need to resolve and store it. We add a helper on `ChatIntegrationService` and switch the controller to use it.

- [ ] **Step 1: Find the existing append site**

Run: `grep -n "credentialId" packages/cli/src/modules/agents/agents.controller.ts | head`
Look around lines 575–591 (per the architecture report) for the `agent.integrations = [...existing, { type, credentialId }]` pattern.

- [ ] **Step 2: Write failing test for the helper**

Append to `packages/cli/src/modules/agents/integrations/__tests__/chat-integration.service.test.ts` (create if missing — there are existing tests for the bridge but not the service helpers):

```typescript
describe('ChatIntegrationService.appendIntegration', () => {
	it('persists credentialName resolved from the credentials repository', async () => {
		const credentialsRepo = mock<CredentialsRepository>();
		credentialsRepo.findOneByOrFail.mockResolvedValue(
			mock<CredentialsEntity>({ id: 'cred-1', name: 'Acme Slack' }),
		);
		const agentRepo = mock<AgentRepository>();
		const agent = mock<Agent>({ id: 'agent-1', integrations: [] });
		agentRepo.save.mockImplementation(async (a) => a);

		const service = new ChatIntegrationService(
			/* registry */ mock(),
			credentialsRepo,
			agentRepo,
			/* logger, etc. — fill from existing constructor */
		);

		const updated = await service.appendIntegration(agent, 'slack', 'cred-1');

		expect(updated.integrations).toEqual([
			{ type: 'slack', credentialId: 'cred-1', credentialName: 'Acme Slack' },
		]);
	});

	it('throws when the credential cannot be found', async () => {
		const credentialsRepo = mock<CredentialsRepository>();
		credentialsRepo.findOneByOrFail.mockRejectedValue(new Error('not found'));
		// ... build service ...
		await expect(service.appendIntegration(agent, 'slack', 'missing'))
			.rejects.toThrow(/not found/);
	});
});
```

(Use the existing `mock<T>()` helper; match the constructor signature of `ChatIntegrationService` exactly — read it from the file before writing the test.)

- [ ] **Step 3: Run test to verify it fails**

Run: `pushd packages/cli && pnpm test src/modules/agents/integrations/__tests__/chat-integration.service.test.ts 2>&1 | tail -20 && popd`
Expected: FAIL — `appendIntegration` does not exist.

- [ ] **Step 4: Implement `appendIntegration`**

In `packages/cli/src/modules/agents/integrations/chat-integration.service.ts`, add (near the existing `connect` method):

```typescript
async appendIntegration(
	agent: Agent,
	type: string,
	credentialId: string,
): Promise<Agent> {
	const credential = await this.credentialsRepository.findOneByOrFail({ id: credentialId });
	const next: AgentCredentialIntegration = {
		type,
		credentialId,
		credentialName: credential.name,
	};
	const filtered = (agent.integrations ?? []).filter(
		(i) => !(i.type === type && 'credentialId' in i && i.credentialId === credentialId),
	);
	agent.integrations = [...filtered, next];
	return await this.agentRepository.save(agent);
}
```

(Constructor injection of `credentialsRepository` may already be present — confirm before adding. If not, add it.)

- [ ] **Step 5: Run test to verify it passes**

Run: `pushd packages/cli && pnpm test src/modules/agents/integrations/__tests__/chat-integration.service.test.ts 2>&1 | tail -15 && popd`
Expected: PASS.

- [ ] **Step 6: Switch the controller call site to use `appendIntegration`**

In `packages/cli/src/modules/agents/agents.controller.ts`, replace the inline mutation block (around lines 575–591) with:

```typescript
const updatedAgent = await this.chatIntegrationService.appendIntegration(
	agent,
	type,
	credentialId,
);
```

Adjust the surrounding flow so `updatedAgent` is the new value used for downstream calls (e.g. status response).

- [ ] **Step 7: Update any existing callers / tests that constructed `AgentCredentialIntegration` literals**

Run: `grep -rn "credentialId:" packages/cli/src/modules/agents --include="*.ts" | grep -v test | grep -v ".d.ts"`
For each TS literal that builds an `AgentCredentialIntegration` without `credentialName`, add the field. Use the credential repository or test fixture name as appropriate.

- [ ] **Step 8: Run typecheck for cli**

Run: `pnpm --filter=n8n typecheck 2>&1 | tail -40`
Expected: PASS (the api-types changes from Task 1 should now compile cleanly).

- [ ] **Step 9: Run targeted tests**

Run: `pushd packages/cli && pnpm test src/modules/agents/integrations 2>&1 | tail -30 && popd`
Expected: PASS.

- [ ] **Step 10: Commit**

```bash
git add packages/cli/src/modules/agents/integrations/chat-integration.service.ts \
        packages/cli/src/modules/agents/integrations/__tests__/chat-integration.service.test.ts \
        packages/cli/src/modules/agents/agents.controller.ts
# plus any test/fixture files updated in Step 7
git commit -m "feat(agents): persist credentialName for chat integrations"
```

---

## Task 4: Compose / decompose helpers in `agents.service.ts`

**Files:**
- Modify: `packages/cli/src/modules/agents/agents.service.ts`
- Create: `packages/cli/src/modules/agents/__tests__/agent-config-composition.test.ts`

The JSON-config view must hide the storage split. Two pure helpers do the work:
- `composeJsonConfig(entity)` — entity → unified `AgentJsonConfig` for read paths.
- `decomposeJsonConfig(config)` — `AgentJsonConfig` → `{ schemaConfig, integrations }` for write paths.

- [ ] **Step 1: Write failing tests**

Create `packages/cli/src/modules/agents/__tests__/agent-config-composition.test.ts`:

```typescript
import {
	composeJsonConfig,
	decomposeJsonConfig,
} from '../agents.service';
import type { Agent } from '../entities/agent.entity';
import type { AgentJsonConfig } from '@n8n/api-types';

describe('composeJsonConfig', () => {
	it('returns the schema unchanged when no integrations are stored', () => {
		const agent = {
			schema: { name: 'A', model: 'anthropic/claude', instructions: 'x' },
			integrations: [],
		} as unknown as Agent;
		expect(composeJsonConfig(agent)).toEqual({
			name: 'A',
			model: 'anthropic/claude',
			instructions: 'x',
			integrations: [],
		});
	});

	it('merges integrations from the storage column into the JSON config', () => {
		const agent = {
			schema: { name: 'A', model: 'anthropic/claude', instructions: 'x' },
			integrations: [
				{ type: 'slack', credentialId: 'c1', credentialName: 'Acme' },
			],
		} as unknown as Agent;
		expect(composeJsonConfig(agent).integrations).toEqual([
			{ type: 'slack', credentialId: 'c1', credentialName: 'Acme' },
		]);
	});

	it('returns null-safe behavior when schema is null', () => {
		const agent = { schema: null, integrations: [] } as unknown as Agent;
		expect(composeJsonConfig(agent)).toBeNull();
	});
});

describe('decomposeJsonConfig', () => {
	it('splits integrations away from the schema-storage payload', () => {
		const input: AgentJsonConfig = {
			name: 'A',
			model: 'anthropic/claude',
			instructions: 'x',
			integrations: [
				{ type: 'schedule', active: true, cronExpression: '0 9 * * *', wakeUpPrompt: 'go' },
			],
		};
		const { schemaConfig, integrations } = decomposeJsonConfig(input);
		expect(schemaConfig).not.toHaveProperty('integrations');
		expect(schemaConfig.name).toBe('A');
		expect(integrations).toEqual(input.integrations);
	});

	it('defaults to an empty integrations array when missing', () => {
		const input: AgentJsonConfig = {
			name: 'A',
			model: 'anthropic/claude',
			instructions: 'x',
		};
		const { integrations } = decomposeJsonConfig(input);
		expect(integrations).toEqual([]);
	});
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agent-config-composition.test.ts 2>&1 | tail -20 && popd`
Expected: FAIL — exports do not exist.

- [ ] **Step 3: Implement helpers**

In `packages/cli/src/modules/agents/agents.service.ts`, near the top (after imports), add and export:

```typescript
export function composeJsonConfig(agent: Agent): AgentJsonConfig | null {
	if (!agent.schema) return null;
	return {
		...agent.schema,
		integrations: agent.integrations ?? [],
	};
}

export function decomposeJsonConfig(config: AgentJsonConfig): {
	schemaConfig: Omit<AgentJsonConfig, 'integrations'>;
	integrations: AgentIntegration[];
} {
	const { integrations, ...schemaConfig } = config;
	return { schemaConfig, integrations: integrations ?? [] };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agent-config-composition.test.ts 2>&1 | tail -15 && popd`
Expected: PASS, 5 passing tests.

- [ ] **Step 5: Commit**

```bash
git add packages/cli/src/modules/agents/agents.service.ts \
        packages/cli/src/modules/agents/__tests__/agent-config-composition.test.ts
git commit -m "feat(agents): add compose/decompose helpers for AgentJsonConfig"
```

---

## Task 5: Use `composeJsonConfig` on read paths

**Files:**
- Modify: `packages/cli/src/modules/agents/agents.service.ts`
- Modify: `packages/cli/src/modules/agents/__tests__/agents-service.test.ts` (add new test)

The two read paths the builder consumes are `getConfig(agentId, projectId)` (returns the JSON config) and `findById(agentId, projectId)` (returns the entity, used by FE in some flows). For now we only change `getConfig` — the entity-shaped response is left for a future cleanup.

- [ ] **Step 1: Write failing test**

Add to the existing `agents-service.test.ts` (or create one if absent) inside a `describe('getConfig')` block:

```typescript
it('includes the integrations stored on the entity', async () => {
	const entity = mock<Agent>({
		schema: { name: 'A', model: 'anthropic/claude', instructions: 'x' },
		integrations: [{ type: 'slack', credentialId: 'c1', credentialName: 'Acme' }],
	});
	agentRepository.findByIdAndProjectId.mockResolvedValue(entity);

	const config = await agentsService.getConfig('agent-1', 'project-1');

	expect(config?.integrations).toEqual([
		{ type: 'slack', credentialId: 'c1', credentialName: 'Acme' },
	]);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agents-service.test.ts -t "getConfig" 2>&1 | tail -20 && popd`
Expected: FAIL — `config.integrations` is `undefined`.

- [ ] **Step 3: Update `getConfig` to compose**

In `agents.service.ts`, replace the body of `getConfig`:

```typescript
async getConfig(
	agentId: string,
	projectId: string,
): Promise<AgentJsonConfig | null> {
	const entity = await this.agentRepository.findByIdAndProjectId(agentId, projectId);
	return composeJsonConfig(entity);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agents-service.test.ts -t "getConfig" 2>&1 | tail -15 && popd`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/cli/src/modules/agents/agents.service.ts \
        packages/cli/src/modules/agents/__tests__/agents-service.test.ts
git commit -m "feat(agents): include integrations in AgentJsonConfig responses"
```

---

## Task 6: Implement integration sync (diff + lifecycle calls)

**Files:**
- Modify: `packages/cli/src/modules/agents/agents.service.ts`
- Modify: `packages/cli/src/modules/agents/integrations/agent-schedule.service.ts`
- Modify: `packages/cli/src/modules/agents/integrations/chat-integration.service.ts`
- Create: `packages/cli/src/modules/agents/__tests__/integration-sync.test.ts`

When the builder rewrites `integrations`, the live runtime state must follow:
- Schedule trigger added/changed/active → register or refresh the cron job.
- Schedule trigger removed or deactivated → stop the cron job.
- Chat integration added → `chatIntegrationService.connect`.
- Chat integration removed → `chatIntegrationService.disconnect`.
- Chat integration changed (different credential) → disconnect old + connect new.

We isolate this in a private `syncIntegrations(savedAgent, oldIntegrations, newIntegrations)` on `AgentsService`.

- [ ] **Step 1: Add idempotent helpers on the schedule service**

In `agent-schedule.service.ts`, add (do not remove existing methods):

```typescript
async applyConfig(agent: Agent): Promise<void> {
	const schedule = this.getScheduleIntegration(agent);
	if (schedule?.active) {
		await this.registerOrRefresh(agent);
	} else {
		this.stop(agent.id);
	}
}
```

`stop()` already exists internally — expose it as `private stop(agentId)` if needed; reuse the cron map.

- [ ] **Step 2: Add idempotent helper on the chat integration service**

In `chat-integration.service.ts`, add:

```typescript
async syncToConfig(
	agent: Agent,
	previous: AgentCredentialIntegration[],
	next: AgentCredentialIntegration[],
): Promise<void> {
	const key = (i: AgentCredentialIntegration) => `${i.type}:${i.credentialId}`;
	const previousKeys = new Set(previous.map(key));
	const nextKeys = new Set(next.map(key));

	for (const integration of previous) {
		if (!nextKeys.has(key(integration))) {
			await this.disconnect(agent, integration.type, integration.credentialId).catch((err) =>
				this.logger.warn('Failed to disconnect chat integration during sync', { err }),
			);
		}
	}
	for (const integration of next) {
		if (!previousKeys.has(key(integration))) {
			await this.connect(
				agent.id,
				integration.credentialId,
				integration.type,
				/* userId */ undefined,
				agent.projectId,
			).catch((err) =>
				this.logger.warn('Failed to connect chat integration during sync', { err }),
			);
		}
	}
}
```

> **Why not throw on sync failure?** Builder writes happen during a chat turn. A platform outage shouldn't roll back the config. We log a warning; the existing status endpoint (`/agents/:id/integrations/status`) already surfaces failures to the FE. Track with telemetry in a follow-up.

- [ ] **Step 3: Write failing test for `AgentsService.syncIntegrations`**

Create `packages/cli/src/modules/agents/__tests__/integration-sync.test.ts`:

```typescript
import { mock } from 'jest-mock-extended';
import { AgentsService } from '../agents.service';
import { AgentScheduleService } from '../integrations/agent-schedule.service';
import { ChatIntegrationService } from '../integrations/chat-integration.service';
import type { Agent } from '../entities/agent.entity';
import type { AgentIntegration } from '@n8n/api-types';

describe('AgentsService.syncIntegrations', () => {
	const buildService = () => {
		const schedule = mock<AgentScheduleService>();
		const chat = mock<ChatIntegrationService>();
		const service = new AgentsService(
			/* fill all constructor deps from the actual class — mock them */
			schedule,
			chat,
			// ...
		);
		return { service, schedule, chat };
	};

	it('routes schedule changes to AgentScheduleService.applyConfig', async () => {
		const { service, schedule } = buildService();
		const agent = mock<Agent>({ id: 'a1' });
		const oldIntegrations: AgentIntegration[] = [];
		const newIntegrations: AgentIntegration[] = [
			{ type: 'schedule', active: true, cronExpression: '0 9 * * *', wakeUpPrompt: 'go' },
		];

		await service['syncIntegrations'](agent, oldIntegrations, newIntegrations);

		expect(schedule.applyConfig).toHaveBeenCalledWith(agent);
	});

	it('passes the credential-integration diff to ChatIntegrationService.syncToConfig', async () => {
		const { service, chat } = buildService();
		const agent = mock<Agent>({ id: 'a1', projectId: 'p1' });
		const oldIntegrations: AgentIntegration[] = [
			{ type: 'slack', credentialId: 'c1', credentialName: 'Acme' },
		];
		const newIntegrations: AgentIntegration[] = [
			{ type: 'slack', credentialId: 'c2', credentialName: 'Acme 2' },
		];

		await service['syncIntegrations'](agent, oldIntegrations, newIntegrations);

		expect(chat.syncToConfig).toHaveBeenCalledWith(
			agent,
			oldIntegrations,
			newIntegrations,
		);
	});

	it('calls applyConfig even when only the schedule active flag flips', async () => {
		const { service, schedule } = buildService();
		const agent = mock<Agent>({ id: 'a1' });
		const oldIntegrations: AgentIntegration[] = [
			{ type: 'schedule', active: true, cronExpression: '0 9 * * *', wakeUpPrompt: 'go' },
		];
		const newIntegrations: AgentIntegration[] = [
			{ type: 'schedule', active: false, cronExpression: '0 9 * * *', wakeUpPrompt: 'go' },
		];

		await service['syncIntegrations'](agent, oldIntegrations, newIntegrations);

		expect(schedule.applyConfig).toHaveBeenCalledWith(agent);
	});
});
```

- [ ] **Step 4: Run test to verify it fails**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/integration-sync.test.ts 2>&1 | tail -20 && popd`
Expected: FAIL — `syncIntegrations` does not exist.

- [ ] **Step 5: Implement `syncIntegrations`**

In `agents.service.ts`, add the private method:

```typescript
import { isAgentScheduleIntegration } from './integrations/agent-schedule.service';

// ...

private async syncIntegrations(
	agent: Agent,
	previous: AgentIntegration[],
	next: AgentIntegration[],
): Promise<void> {
	// Schedule: any change in presence/cron/active triggers re-application.
	const prevSchedule = previous.find(isAgentScheduleIntegration);
	const nextSchedule = next.find(isAgentScheduleIntegration);
	if (!isEqual(prevSchedule, nextSchedule)) {
		await this.scheduleService.applyConfig(agent);
	}

	// Chat: delegate diff to the chat service.
	const prevChat = previous.filter(
		(i): i is AgentCredentialIntegration => !isAgentScheduleIntegration(i),
	);
	const nextChat = next.filter(
		(i): i is AgentCredentialIntegration => !isAgentScheduleIntegration(i),
	);
	await this.chatIntegrationService.syncToConfig(agent, prevChat, nextChat);
}
```

(Use `lodash/isEqual` if available, otherwise a small inline structural compare on `cronExpression`, `wakeUpPrompt`, `active`.)

- [ ] **Step 6: Run test to verify it passes**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/integration-sync.test.ts 2>&1 | tail -15 && popd`
Expected: PASS, 3 passing tests.

- [ ] **Step 7: Commit**

```bash
git add packages/cli/src/modules/agents/agents.service.ts \
        packages/cli/src/modules/agents/integrations/agent-schedule.service.ts \
        packages/cli/src/modules/agents/integrations/chat-integration.service.ts \
        packages/cli/src/modules/agents/__tests__/integration-sync.test.ts
git commit -m "feat(agents): sync runtime triggers after JSON config writes"
```

---

## Task 7: Wire decompose + sync into `updateConfig`

**Files:**
- Modify: `packages/cli/src/modules/agents/agents.service.ts`
- Modify: `packages/cli/src/modules/agents/__tests__/agents-service.test.ts`

This closes the write loop. `updateConfig` now decomposes the inbound JSON config, persists the integrations column, and triggers `syncIntegrations`.

- [ ] **Step 1: Write failing test**

Add to `agents-service.test.ts` inside `describe('updateConfig')`:

```typescript
it('persists integrations from the JSON config to the entity column', async () => {
	const entity = mock<Agent>({
		schema: { name: 'A', model: 'anthropic/claude', instructions: 'x' },
		integrations: [],
	});
	agentRepository.findByIdAndProjectId.mockResolvedValue(entity);
	agentRepository.save.mockImplementation(async (a) => a);

	const config: AgentJsonConfig = {
		name: 'A',
		model: 'anthropic/claude',
		instructions: 'x',
		integrations: [
			{ type: 'slack', credentialId: 'c1', credentialName: 'Acme' },
		],
	};

	await agentsService.updateConfig('agent-1', 'project-1', config);

	expect(entity.integrations).toEqual(config.integrations);
	expect(entity.schema).not.toHaveProperty('integrations');
});

it('invokes syncIntegrations with the previous and next integration arrays', async () => {
	const previous = [{ type: 'slack', credentialId: 'c0', credentialName: 'Old' }];
	const entity = mock<Agent>({
		schema: { name: 'A', model: 'anthropic/claude', instructions: 'x' },
		integrations: previous,
	});
	agentRepository.findByIdAndProjectId.mockResolvedValue(entity);
	agentRepository.save.mockImplementation(async (a) => a);
	const syncSpy = jest.spyOn(agentsService as any, 'syncIntegrations').mockResolvedValue(undefined);

	const next = [{ type: 'slack', credentialId: 'c1', credentialName: 'New' }];
	await agentsService.updateConfig('agent-1', 'project-1', {
		name: 'A',
		model: 'anthropic/claude',
		instructions: 'x',
		integrations: next,
	});

	expect(syncSpy).toHaveBeenCalledWith(entity, previous, next);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agents-service.test.ts -t "updateConfig" 2>&1 | tail -25 && popd`
Expected: FAIL — `entity.integrations` is unchanged or sync not called.

- [ ] **Step 3: Update `updateConfig`**

Replace the body of `updateConfig` in `agents.service.ts`:

```typescript
async updateConfig(
	agentId: string,
	projectId: string,
	config: AgentJsonConfig,
): Promise<{ config: AgentJsonConfig; updatedAt: Date; versionId: string | null }> {
	const entity = await this.agentRepository.findByIdAndProjectId(agentId, projectId);
	const result = await this.validateConfig(config);
	const { schemaConfig, integrations: nextIntegrations } = decomposeJsonConfig(result.config);
	const previousIntegrations = entity.integrations ?? [];

	entity.schema = schemaConfig as AgentJsonConfig;
	entity.name = schemaConfig.name;
	entity.integrations = nextIntegrations;
	this.markDraftDirty(entity);
	this.clearRuntimes(agentId);
	const saved = await this.agentRepository.save(entity);

	await this.syncIntegrations(saved, previousIntegrations, nextIntegrations);

	return {
		config: composeJsonConfig(saved)!,
		updatedAt: saved.updatedAt,
		versionId: saved.versionId,
	};
}
```

> **Type note on `entity.schema`:** `agent.schema` is typed as `AgentJsonConfig | null`. `schemaConfig` here is `Omit<AgentJsonConfig, 'integrations'>`. Since `integrations` is optional on `AgentJsonConfig`, the assignment is structurally safe — but the cast (`as AgentJsonConfig`) is intentional and avoids a wider type change. Long-term we may split the storage type.

- [ ] **Step 4: Run test to verify it passes**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agents-service.test.ts -t "updateConfig" 2>&1 | tail -15 && popd`
Expected: PASS.

- [ ] **Step 5: Run all agents-service tests for regression**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agents-service.test.ts 2>&1 | tail -25 && popd`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/cli/src/modules/agents/agents.service.ts \
        packages/cli/src/modules/agents/__tests__/agents-service.test.ts
git commit -m "feat(agents): apply integrations changes on JSON config update"
```

---

## Task 8: Add `list_integration_types` builder tool

**Files:**
- Modify: `packages/cli/src/modules/agents/builder/agents-builder-tools.service.ts`
- Create: `packages/cli/src/modules/agents/builder/__tests__/list-integration-types.tool.test.ts`

The builder LLM needs a discovery tool to learn what trigger types exist. The data source is `ChatIntegrationRegistry` (Slack, Telegram, Linear) plus a static schedule entry.

- [ ] **Step 1: Write failing test**

Create `packages/cli/src/modules/agents/builder/__tests__/list-integration-types.tool.test.ts`:

```typescript
import { mock } from 'jest-mock-extended';
import { ChatIntegrationRegistry } from '../../integrations/agent-chat-integration';
import { AgentsBuilderToolsService } from '../agents-builder-tools.service';

describe('list_integration_types tool', () => {
	it('returns the schedule trigger plus every registered chat integration', async () => {
		const registry = mock<ChatIntegrationRegistry>();
		registry.list.mockReturnValue([
			{
				type: 'slack',
				displayLabel: 'Slack',
				displayIcon: 'slack',
				credentialTypes: ['slackApi', 'slackOAuth2Api'],
			} as never,
			{
				type: 'telegram',
				displayLabel: 'Telegram',
				displayIcon: 'telegram',
				credentialTypes: ['telegramBotToken'],
			} as never,
		]);
		const service = new AgentsBuilderToolsService(
			/* fill remaining deps from the actual constructor */
			registry,
		);

		const tool = service.getListIntegrationTypesTool();
		const result = await tool.execute({});

		expect(result).toEqual([
			{ type: 'schedule', label: 'Schedule', icon: 'clock', credentialTypes: [] },
			{ type: 'slack', label: 'Slack', icon: 'slack', credentialTypes: ['slackApi', 'slackOAuth2Api'] },
			{ type: 'telegram', label: 'Telegram', icon: 'telegram', credentialTypes: ['telegramBotToken'] },
		]);
	});
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pushd packages/cli && pnpm test src/modules/agents/builder/__tests__/list-integration-types.tool.test.ts 2>&1 | tail -20 && popd`
Expected: FAIL — `getListIntegrationTypesTool` does not exist.

- [ ] **Step 3: Implement the tool**

In `agents-builder-tools.service.ts`, follow the pattern of the other tools (e.g. `getSearchNodesTool`). Add:

```typescript
getListIntegrationTypesTool() {
	return tool({
		name: 'list_integration_types',
		description:
			'List trigger / integration types that can be added to the agent\'s `integrations` array. ' +
			'Returns the schedule trigger plus every connected chat platform with its credential type(s). ' +
			'Use this BEFORE asking the user for a credential — pass the returned `credentialTypes` to `ask_credential`.',
		parameters: z.object({}),
		execute: async () => {
			const chat = this.chatIntegrationRegistry.list().map((i) => ({
				type: i.type,
				label: i.displayLabel,
				icon: i.displayIcon,
				credentialTypes: i.credentialTypes,
			}));
			return [
				{ type: 'schedule', label: 'Schedule', icon: 'clock', credentialTypes: [] },
				...chat,
			];
		},
	});
}
```

Then register the tool in the same service's `getBuilderTools()` (or whatever method assembles the tools list passed to the builder Agent) by adding `this.getListIntegrationTypesTool()` to the returned record.

- [ ] **Step 4: Run test to verify it passes**

Run: `pushd packages/cli && pnpm test src/modules/agents/builder/__tests__/list-integration-types.tool.test.ts 2>&1 | tail -15 && popd`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/cli/src/modules/agents/builder/agents-builder-tools.service.ts \
        packages/cli/src/modules/agents/builder/__tests__/list-integration-types.tool.test.ts
git commit -m "feat(agents): add list_integration_types builder tool"
```

---

## Task 9: Update builder system prompt

**Files:**
- Modify: `packages/cli/src/modules/agents/builder/agents-builder-prompts.ts`
- Modify: `packages/cli/src/modules/agents/__tests__/agents-builder-prompts.test.ts`

The schema reference (auto-generated) will already include `integrations`, but the LLM needs explicit guidance on **when** and **how** to use it. We add a new section, mirror the structure of `TOOL_TYPES_SECTION`, and wire it into the assembled prompt.

- [ ] **Step 1: Write failing test**

Append to `agents-builder-prompts.test.ts`:

```typescript
describe('INTEGRATIONS_SECTION', () => {
	it('appears in the assembled system prompt', () => {
		const prompt = buildBuilderSystemPrompt({
			builderModel: 'anthropic/claude',
			currentConfig: null,
			customTools: {},
		});
		expect(prompt).toContain('## Integrations (triggers)');
		expect(prompt).toContain('list_integration_types');
		expect(prompt).toContain('schedule trigger');
	});

	it('describes the credential workflow for chat integrations', () => {
		const prompt = buildBuilderSystemPrompt({
			builderModel: 'anthropic/claude',
			currentConfig: null,
			customTools: {},
		});
		expect(prompt).toContain('ask_credential');
		expect(prompt).toContain('credentialName');
	});
});
```

(Adjust `buildBuilderSystemPrompt` and parameter shape to match the actual export — read the existing test for reference.)

- [ ] **Step 2: Run test to verify it fails**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agents-builder-prompts.test.ts -t "INTEGRATIONS_SECTION" 2>&1 | tail -20 && popd`
Expected: FAIL — section not present.

- [ ] **Step 3: Add the new section constant**

In `agents-builder-prompts.ts`, add:

```typescript
export const INTEGRATIONS_SECTION = `
## Integrations (triggers)

The \`integrations\` field on the agent JSON config defines how an agent gets triggered.
There are two kinds of triggers:

1. **Schedule trigger** — runs the agent on a cron schedule. Singleton per agent.
   Shape:
   \`\`\`json
   { "type": "schedule", "active": true, "cronExpression": "0 9 * * *", "wakeUpPrompt": "Daily standup ping" }
   \`\`\`
   - \`cronExpression\`: standard cron syntax (5 fields).
   - \`active\`: false until the agent is published; flip to true once you want it firing.
   - \`wakeUpPrompt\`: the message the agent receives when fired.

2. **Chat integrations** — connect the agent to a messaging platform. Multiple are allowed.
   Shape:
   \`\`\`json
   { "type": "slack", "credentialId": "<id>", "credentialName": "<name>" }
   \`\`\`

### Workflow for adding integrations

1. Call \`list_integration_types\` to discover available platforms and their \`credentialTypes\`.
2. For chat integrations, call \`ask_credential\` with the matching credential types — this returns \`{ id, name }\`.
3. Use \`patch_config\` (or \`write_config\`) to add an entry to the \`integrations\` array.
   - For chat integrations, set both \`credentialId\` and \`credentialName\` from the \`ask_credential\` result.
   - For the schedule trigger, write the cron expression directly (no credential required).

Never invent credential IDs. Always go through \`ask_credential\`.
`;
```

Then, in the function that assembles the prompt (`buildBuilderSystemPrompt` or the equivalent — confirm name from the file), insert `INTEGRATIONS_SECTION` after `TOOL_TYPES_SECTION`.

- [ ] **Step 4: Run test to verify it passes**

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agents-builder-prompts.test.ts -t "INTEGRATIONS_SECTION" 2>&1 | tail -15 && popd`
Expected: PASS.

- [ ] **Step 5: Verify the auto-generated schema text now includes integrations**

Add this assertion to the existing schema-reference test (or write a new one):

```typescript
it('includes the integrations field in the schema reference', () => {
	const prompt = buildBuilderSystemPrompt({
		builderModel: 'anthropic/claude',
		currentConfig: null,
		customTools: {},
	});
	expect(prompt).toContain('integrations?:');
	expect(prompt).toContain('"slack"');
});
```

Run: `pushd packages/cli && pnpm test src/modules/agents/__tests__/agents-builder-prompts.test.ts 2>&1 | tail -20 && popd`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/cli/src/modules/agents/builder/agents-builder-prompts.ts \
        packages/cli/src/modules/agents/__tests__/agents-builder-prompts.test.ts
git commit -m "feat(agents): document integrations management in builder prompt"
```

---

## Task 10: Full-package verification + lint

**Files:** None (verification only).

- [ ] **Step 1: Run typecheck across cli**

Run: `pnpm --filter=n8n typecheck 2>&1 | tail -30`
Expected: PASS.

- [ ] **Step 2: Run typecheck across api-types**

Run: `pnpm --filter=@n8n/api-types typecheck`
Expected: PASS.

- [ ] **Step 3: Run lint on the agents module**

Run: `pushd packages/cli && pnpm lint --quiet src/modules/agents 2>&1 | tail -30 && popd`
Expected: PASS.

- [ ] **Step 4: Run all agent-related tests**

Run: `pushd packages/cli && pnpm test src/modules/agents 2>&1 | tail -30 && popd`
Expected: PASS.

- [ ] **Step 5: Commit any minor lint fixes**

If lint required formatting fixes, commit them:

```bash
git add -A packages/cli/src/modules/agents
git commit -m "chore: fix lint in agents module"
```

If no fixes were needed, skip this step.

---

## Task 11: Manual smoke check via builder chat

**Files:** None (manual verification).

This is a sanity pass. The dev server is running at `http://localhost:8080` (frontend hot-reload) per the user's environment notes. Run a real builder conversation to confirm the new behavior.

- [ ] **Step 1: Verify dev server is up**

Open `http://localhost:8080`. If it isn't running, start it per the project's dev workflow.

- [ ] **Step 2: Create a fresh agent via the UI**

Navigate to Agents → Create new. Open the builder chat.

- [ ] **Step 3: Ask the builder to add a Slack integration**

Prompt: "Connect this agent to my Slack workspace."
Expected:
- Builder calls `list_integration_types`.
- Builder calls `ask_credential` with `slackApi` / `slackOAuth2Api`.
- After picking a credential, builder applies a `patch_config` adding `{ type: 'slack', credentialId, credentialName }` to `integrations`.
- The agent's integrations status endpoint reflects the new connection.

- [ ] **Step 4: Ask the builder to add a daily schedule**

Prompt: "Run this agent every weekday at 9am with prompt 'Run the daily check.'"
Expected:
- Builder writes `{ type: 'schedule', active: false, cronExpression: '0 9 * * 1-5', wakeUpPrompt: '...' }`.
- Active stays false (agent isn't published yet — this matches the existing constraint).

- [ ] **Step 5: Read back the JSON config**

`GET /agents/:id/config` should return the integrations field populated.

- [ ] **Step 6: Verify the storage column matches**

Query the DB or use existing endpoints:
- `agent.integrations` column has the same array.
- `agent.schema` does **not** have an `integrations` field (decomposition worked).

If anything is off, capture findings in a follow-up task. Otherwise, this plan is complete.

---

## Notes for the implementer

- **Avoid `as any` / `as unknown as T`.** Where the existing codebase uses `mock<T>()` from `jest-mock-extended`, prefer that.
- **Keep tool handlers idempotent.** `chatIntegrationService.syncToConfig` may run multiple times for the same final state across rapid builder edits. Connecting an already-connected `(agentId, type, credentialId)` should be a no-op or short-circuit.
- **Do not modify `AgentPublishedVersion`.** Out of scope. If schedule activation needs the published snapshot to include integrations, raise it as a follow-up.
- **i18n.** No new user-visible strings are introduced by this change (builder prompt text is internal). If you find any UI string changes required, route them through `@n8n/i18n` per the AGENTS.md rule.
- **Telemetry hooks.** The Linear A/C calls for telemetry events. This plan leaves placement up to a follow-up because the events haven't been specified yet — flag in PR description.
- **Scaling-mode A/C** (queue-mode, multi-main): the existing `ChatIntegrationService.reconnectAll` and `AgentScheduleService.reconnectAll` already run on startup. As long as those still read `agent.integrations` (which they do — we didn't move the column), multi-main behavior is unchanged. Note this in the PR description; if the QA needs explicit verification, plan a follow-up test with a queue-mode container.
