# HelmsMan — Step-by-Step Build Guide (NitroStack IDE)

**Autonomous Kubernetes Remediation · MCP server on NitroStack + NitroStudio**

Follow phases 0 → 8 top to bottom. Every command and every IDE click-path is spelled out.
Boxes `[ ]` are a checklist. Total realistic time: ~4–6 hours for a working demo.

> **The one idea that wins this:** the Availability Guardian's veto is not a hardcoded label — it is
> read live from `PodDisruptionBudget.status` on a real cluster, and the server **refuses unsafe
> scale-downs in code**. Prove that in NitroStudio and you've beaten every "simulated" project.

---

## Phase 0 — Prerequisites (30 min)

Install these once. Verify each with the check command.

| Tool | Install | Verify |
|---|---|---|
| Node.js ≥ 18 | https://nodejs.org | `node -v` |
| Docker Desktop | https://docker.com | `docker ps` |
| kind | `go install sigs.k8s.io/kind@latest` **or** download binary | `kind version` |
| kubectl | https://kubernetes.io/docs/tasks/tools/ | `kubectl version --client` |
| NitroStack CLI | `npm install -g @nitrostack/cli` | `nitrostack-cli --help` |
| NitroStudio (desktop app) | https://nitrostack.ai/studio | app launches |
| An LLM API key | OpenAI **or** Gemini (Studio's AI chat needs one) | key in hand |

```
[ ] node -v            -> v18+ 
[ ] docker ps          -> no error (daemon running)
[ ] kind version       -> prints version
[ ] kubectl version --client
[ ] nitrostack-cli --help
[ ] NitroStudio opens
[ ] OpenAI/Gemini API key ready
```

---

## Phase 1 — Stand up the live cluster (15 min)

The cluster is your "real data source." Create it and deploy the intentionally over-provisioned
workload guarded by a PodDisruptionBudget.

```bash
kind create cluster --name helmsman
kubectl create namespace shop
```

Create `manifests/demo.yaml` (anywhere — e.g. a `helmsman-cluster/` folder):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-service
  namespace: shop
  labels: { app: checkout, tier: frontend }
spec:
  replicas: 5
  selector: { matchLabels: { app: checkout } }
  template:
    metadata: { labels: { app: checkout } }
    spec:
      containers:
        - name: web
          image: nginx:1.27-alpine
          resources:
            requests: { cpu: "250m", memory: "128Mi" }   # the FinOps cost signal
            limits:   { cpu: "500m", memory: "256Mi" }
          ports: [ { containerPort: 80 } ]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout-pdb
  namespace: shop
spec:
  minAvailable: 3            # the Availability Guardian's hard floor — enforced by real K8s
  selector: { matchLabels: { app: checkout } }
```

```bash
kubectl apply -f manifests/demo.yaml

# WAIT until disruptionsAllowed is a number > 0 before continuing (PDB status takes a few seconds):
kubectl get pdb -n shop checkout-pdb -o jsonpath='{.status.disruptionsAllowed}{"\n"}'
```

```
[ ] kind cluster "helmsman" created
[ ] kubectl config current-context  -> kind-helmsman
[ ] kubectl get pods -n shop        -> 5 pods Running
[ ] kubectl get pdb -n shop         -> ALLOWED DISRUPTIONS is a number (e.g. 2)
```

If `current-context` is not `kind-helmsman`: `kubectl config use-context kind-helmsman`.

---

## Phase 2 — Scaffold the NitroStack project (15 min)

```bash
# From your hackathon folder:
nitrostack-cli init helmsman --template typescript
cd helmsman
npm install
npm install @kubernetes/client-node     # the only extra dependency
```

Generated layout (yours may vary slightly):

```
helmsman/
├── src/
│   ├── modules/        # <- your feature module goes here
│   ├── app.module.ts   # root module
│   └── index.ts        # entry point
├── widgets/
├── .env.example
└── package.json
```

**Pin the Kubernetes client major version now — this is the #1 footgun.** In `package.json` confirm:

```json
"dependencies": {
  "@kubernetes/client-node": "^1.0.0"
}
```

```bash
npm ls @kubernetes/client-node    # MUST show 1.x. If it shows 0.x, this guide's code won't match.
```

> This guide's code targets client-node **1.x** (methods take an object param and return the body
> directly). On 0.x, calls use positional args and return `{ body }` — don't mix.

Ensure `tsconfig.json` has decorator support (NitroStack needs it):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "outDir": "dist",
    "strict": true,
    "skipLibCheck": true
  }
}
```

```
[ ] project created, npm install clean
[ ] @kubernetes/client-node is 1.x
[ ] tsconfig has experimentalDecorators + emitDecoratorMetadata
```

---

## Phase 3 — Create the module files (60–90 min)

Create the folder `src/modules/k8s/` and add the 9 files below **exactly**. This is the whole server.

> Tip: you can scaffold empty primitives with the CLI to get the wiring, then paste the bodies:
> `nitrostack-cli generate tool k8s` / `generate resource k8s` / `generate prompt k8s`.
> Or just create the files by hand as below.

### 3.1 `src/modules/k8s/client.ts` — shared cluster client

```ts
import * as k8s from '@kubernetes/client-node';

const kc = new k8s.KubeConfig();
kc.loadFromDefault();                       // reads ~/.kube/config that kind wrote

export const core   = kc.makeApiClient(k8s.CoreV1Api);
export const apps   = kc.makeApiClient(k8s.AppsV1Api);
export const policy = kc.makeApiClient(k8s.PolicyV1Api);
export const kubeConfig = kc;
```

### 3.2 `src/modules/k8s/policy.data.ts` — business policy layer

```ts
export const OPS_POLICY = {
  minReplicas: {
    'shop/checkout-service': 3,
  } as Record<string, number>,
  cost: { inrPerVcpuHour: 3.4, inrPerGiBHour: 0.9 },
  rightsizeUtilizationTarget: 0.6,
} as const;

export function fqName(ns: string, name: string): string { return `${ns}/${name}`; }
```

### 3.3 `src/modules/k8s/safety.ts` — the pure veto function (the one piece that MUST be correct)

```ts
export interface ScaleContext {
  currentReplicas: number;
  targetReplicas: number;
  minReplicasPolicy: number;
  pdbDesiredHealthy: number;
  pdbDisruptionsAllowed: number;
}
export interface Decision { allowed: boolean; reason: string; }

export function evaluateScale(ctx: ScaleContext): Decision {
  if (ctx.targetReplicas >= ctx.currentReplicas) {
    return { allowed: true, reason: 'Scale up or no-op — always safe.' };
  }
  const floor = Math.max(ctx.minReplicasPolicy, ctx.pdbDesiredHealthy);
  if (ctx.targetReplicas < floor) {
    return { allowed: false,
      reason: `VETO: target ${ctx.targetReplicas} < required floor ${floor} ` +
              `(policy min ${ctx.minReplicasPolicy}, PDB desiredHealthy ${ctx.pdbDesiredHealthy}).` };
  }
  const podsRemoved = ctx.currentReplicas - ctx.targetReplicas;
  if (podsRemoved > ctx.pdbDisruptionsAllowed) {
    return { allowed: false,
      reason: `VETO: removing ${podsRemoved} pods exceeds PDB disruptionsAllowed ` +
              `${ctx.pdbDisruptionsAllowed} right now — would breach the budget.` };
  }
  return { allowed: true, reason: `Safe: stays at/above floor ${floor}, within disruption budget.` };
}
```

### 3.4 `src/modules/k8s/safety.test.ts` — runnable self-check (run BEFORE touching the cluster)

```ts
import assert from 'node:assert';
import { evaluateScale } from './safety.js';

const base = { minReplicasPolicy: 3, pdbDesiredHealthy: 3, pdbDisruptionsAllowed: 2 };

assert.strictEqual(evaluateScale({ ...base, currentReplicas: 5, targetReplicas: 1 }).allowed, false);
assert.strictEqual(evaluateScale({ ...base, currentReplicas: 5, targetReplicas: 3 }).allowed, true);
assert.strictEqual(
  evaluateScale({ minReplicasPolicy: 0, pdbDesiredHealthy: 2, pdbDisruptionsAllowed: 2,
                  currentReplicas: 6, targetReplicas: 2 }).allowed, false);
assert.strictEqual(evaluateScale({ ...base, currentReplicas: 3, targetReplicas: 5 }).allowed, true);

console.log('safety self-check: OK');
```

### 3.5 `src/modules/k8s/audit.ts` — in-memory audit log

```ts
export interface ActionRecord {
  timestamp: string; action: string; target: string;
  detail: Record<string, unknown>;
  proposedBy: 'FinOpsAgent' | 'AvailabilityGuardian' | 'Operator';
  status: 'EXECUTED' | 'REJECTED'; reason?: string;
}
const log: ActionRecord[] = [];   // warm-instance lifetime; swap to KV if durability needed
export const recordAction = (r: ActionRecord) => { log.push(r); };
export const getActionLog = () => log;
```

### 3.6 `src/modules/k8s/k8s.tools.ts` — the 6 tools (incl. the server-side safety gate)

```ts
import { ToolDecorator as Tool, z, ExecutionContext } from '@nitrostack/core';
import { core, apps, policy } from './client.js';
import { OPS_POLICY, fqName } from './policy.data.js';
import { evaluateScale } from './safety.js';
import { recordAction, getActionLog } from './audit.js';

function cpuToVcpu(req?: string): number {
  if (!req) return 0;
  return req.endsWith('m') ? parseInt(req) / 1000 : parseFloat(req);
}
function memToGiB(req?: string): number {
  if (!req) return 0;
  if (req.endsWith('Mi')) return parseInt(req) / 1024;
  if (req.endsWith('Gi')) return parseFloat(req);
  return 0;
}

export class K8sTools {
  @Tool({
    name: 'get_cluster_state',
    description:
      'Snapshot of the live cluster: nodes, and every deployment with replicas, resource requests, ' +
      'and estimated hourly cost (₹) from requested CPU/memory × replicas. The FinOps agent\'s read.',
    inputSchema: z.object({ namespace: z.string().optional().default('shop') }),
  })
  async getClusterState(input: { namespace?: string }, ctx: ExecutionContext) {
    const ns = input.namespace ?? 'shop';
    const nodeList = await core.listNode();
    const depList = await apps.listNamespacedDeployment({ namespace: ns });

    const nodes = nodeList.items.map((n) => ({
      name: n.metadata?.name,
      unschedulable: n.spec?.unschedulable ?? false,
      allocatableCpu: n.status?.allocatable?.['cpu'],
      allocatableMem: n.status?.allocatable?.['memory'],
    }));

    const deployments = depList.items.map((d) => {
      const replicas = d.spec?.replicas ?? 0;
      const c = d.spec?.template?.spec?.containers?.[0];
      const vcpu = cpuToVcpu(c?.resources?.requests?.['cpu']);
      const gib = memToGiB(c?.resources?.requests?.['memory']);
      const costPerHr =
        replicas * (vcpu * OPS_POLICY.cost.inrPerVcpuHour + gib * OPS_POLICY.cost.inrPerGiBHour);
      return {
        name: d.metadata?.name, namespace: ns, replicas,
        readyReplicas: d.status?.readyReplicas ?? 0,
        requestPerPod: { vcpu, gib },
        estCostInrPerHr: Number(costPerHr.toFixed(2)),
        minReplicasPolicy: OPS_POLICY.minReplicas[fqName(ns, d.metadata?.name ?? '')] ?? 1,
      };
    });
    return { namespace: ns, nodes, deployments, totalDeployments: deployments.length };
  }

  @Tool({
    name: 'get_pod_health',
    description: 'Per-pod phase, readiness, and restart counts for a namespace — the availability read.',
    inputSchema: z.object({ namespace: z.string().optional().default('shop') }),
  })
  async getPodHealth(input: { namespace?: string }, ctx: ExecutionContext) {
    const ns = input.namespace ?? 'shop';
    const pods = await core.listNamespacedPod({ namespace: ns });
    return {
      namespace: ns,
      pods: pods.items.map((p) => ({
        name: p.metadata?.name, phase: p.status?.phase,
        ready: p.status?.conditions?.find((c) => c.type === 'Ready')?.status === 'True',
        restarts: p.status?.containerStatuses?.reduce((a, cs) => a + (cs.restartCount ?? 0), 0) ?? 0,
        node: p.spec?.nodeName,
      })),
    };
  }

  @Tool({
    name: 'check_disruption_safety',
    description:
      'Read the LIVE PodDisruptionBudget status for a deployment and evaluate whether scaling to a ' +
      'target replica count is safe. What the Availability Guardian calls before approving.',
    inputSchema: z.object({
      name: z.string(), namespace: z.string().optional().default('shop'),
      targetReplicas: z.number().int().min(0),
    }),
  })
  async checkDisruptionSafety(
    input: { name: string; namespace?: string; targetReplicas: number }, ctx: ExecutionContext) {
    const ns = input.namespace ?? 'shop';
    const dep = await apps.readNamespacedDeployment({ name: input.name, namespace: ns });
    const current = dep.spec?.replicas ?? 0;

    const pdbs = await policy.listNamespacedPodDisruptionBudget({ namespace: ns });
    const appLabel = dep.spec?.selector?.matchLabels?.['app'];
    const pdb = pdbs.items.find((b) => b.spec?.selector?.matchLabels?.['app'] === appLabel);

    const decision = evaluateScale({
      currentReplicas: current, targetReplicas: input.targetReplicas,
      minReplicasPolicy: OPS_POLICY.minReplicas[fqName(ns, input.name)] ?? 1,
      pdbDesiredHealthy: pdb?.status?.desiredHealthy ?? 0,
      pdbDisruptionsAllowed: pdb?.status?.disruptionsAllowed ?? 0,
    });

    return {
      deployment: fqName(ns, input.name), currentReplicas: current, targetReplicas: input.targetReplicas,
      pdb: pdb ? {
        name: pdb.metadata?.name, desiredHealthy: pdb.status?.desiredHealthy,
        disruptionsAllowed: pdb.status?.disruptionsAllowed, currentHealthy: pdb.status?.currentHealthy,
      } : null,
      decision,
    };
  }

  @Tool({
    name: 'scale_deployment',
    description:
      'Scale a deployment. SAFETY-ENFORCED: server re-reads the live PDB and policy floor and REJECTS ' +
      'any scale-down that breaches them, regardless of caller. Read-modify-write on scale subresource.',
    inputSchema: z.object({
      name: z.string(), namespace: z.string().optional().default('shop'),
      replicas: z.number().int().min(0), rationale: z.string(),
      proposedBy: z.enum(['FinOpsAgent', 'AvailabilityGuardian', 'Operator']).default('FinOpsAgent'),
    }),
  })
  async scaleDeployment(
    input: { name: string; namespace?: string; replicas: number; rationale: string;
             proposedBy?: 'FinOpsAgent' | 'AvailabilityGuardian' | 'Operator' }, ctx: ExecutionContext) {
    const ns = input.namespace ?? 'shop';

    const safety = await this.checkDisruptionSafety(
      { name: input.name, namespace: ns, targetReplicas: input.replicas }, ctx);
    if (!safety.decision.allowed) {
      const rec = { timestamp: new Date().toISOString(), action: 'scale_deployment',
        target: fqName(ns, input.name), detail: { replicas: input.replicas, pdb: safety.pdb },
        proposedBy: input.proposedBy ?? 'FinOpsAgent', status: 'REJECTED' as const,
        reason: safety.decision.reason };
      recordAction(rec);
      ctx.logger?.warn?.('scale REJECTED', rec);
      return { ok: false, rejected: true, reason: safety.decision.reason, safety };
    }

    const scale = await apps.readNamespacedDeploymentScale({ name: input.name, namespace: ns });
    scale.spec = { replicas: input.replicas };
    await apps.replaceNamespacedDeploymentScale({ name: input.name, namespace: ns, body: scale });

    const rec = { timestamp: new Date().toISOString(), action: 'scale_deployment',
      target: fqName(ns, input.name), detail: { replicas: input.replicas, rationale: input.rationale },
      proposedBy: input.proposedBy ?? 'FinOpsAgent', status: 'EXECUTED' as const };
    recordAction(rec);
    ctx.logger?.info?.('scale EXECUTED', rec);
    return { ok: true, scaledTo: input.replicas, auditLogSize: getActionLog().length };
  }

  @Tool({
    name: 'rollback_deployment',
    description: 'Roll a deployment back to its previous revision (equivalent to `kubectl rollout undo`).',
    inputSchema: z.object({ name: z.string(), namespace: z.string().optional().default('shop') }),
  })
  async rollbackDeployment(input: { name: string; namespace?: string }, ctx: ExecutionContext) {
    const ns = input.namespace ?? 'shop';
    const dep = await apps.readNamespacedDeployment({ name: input.name, namespace: ns });
    const selector = Object.entries(dep.spec?.selector?.matchLabels ?? {})
      .map(([k, v]) => `${k}=${v}`).join(',');
    const rsList = await apps.listNamespacedReplicaSet({ namespace: ns, labelSelector: selector });
    const owned = rsList.items
      .filter((rs) => rs.metadata?.ownerReferences?.some((o) => o.name === input.name))
      .sort((a, b) =>
        Number(b.metadata?.annotations?.['deployment.kubernetes.io/revision'] ?? 0) -
        Number(a.metadata?.annotations?.['deployment.kubernetes.io/revision'] ?? 0));
    const prev = owned[1];
    if (!prev) return { ok: false, reason: 'No previous revision to roll back to.' };
    dep.spec!.template = prev.spec!.template;
    await apps.replaceNamespacedDeployment({ name: input.name, namespace: ns, body: dep });
    recordAction({ timestamp: new Date().toISOString(), action: 'rollback_deployment',
      target: fqName(ns, input.name),
      detail: { toRevision: prev.metadata?.annotations?.['deployment.kubernetes.io/revision'] },
      proposedBy: 'Operator', status: 'EXECUTED' });
    return { ok: true, rolledBackTo: prev.metadata?.annotations?.['deployment.kubernetes.io/revision'] };
  }

  @Tool({
    name: 'cordon_node',
    description: 'Mark a node unschedulable. SAFETY-ENFORCED: refuses if it would strand all workloads.',
    inputSchema: z.object({ node: z.string(), rationale: z.string() }),
  })
  async cordonNode(input: { node: string; rationale: string }, ctx: ExecutionContext) {
    const nodes = await core.listNode();
    const schedulable = nodes.items.filter((n) => !n.spec?.unschedulable);
    const already = nodes.items.find((n) => n.metadata?.name === input.node)?.spec?.unschedulable;
    if (schedulable.length <= 1 && !already) {
      const reason = 'VETO: cordoning the last schedulable node would strand all workloads.';
      recordAction({ timestamp: new Date().toISOString(), action: 'cordon_node',
        target: input.node, detail: {}, proposedBy: 'Operator', status: 'REJECTED', reason });
      return { ok: false, rejected: true, reason };
    }
    const node = await core.readNode({ name: input.node });
    node.spec = { ...node.spec, unschedulable: true };
    await core.replaceNode({ name: input.node, body: node });
    recordAction({ timestamp: new Date().toISOString(), action: 'cordon_node',
      target: input.node, detail: { rationale: input.rationale }, proposedBy: 'Operator', status: 'EXECUTED' });
    return { ok: true, cordoned: input.node };
  }
}
```

### 3.7 `src/modules/k8s/k8s.resources.ts` — the 3 resources

```ts
import { ResourceDecorator as Resource, ExecutionContext } from '@nitrostack/core';
import { core, apps, policy } from './client.js';
import { OPS_POLICY } from './policy.data.js';
import { getActionLog } from './audit.js';

export class K8sResources {
  @Resource({
    uri: 'k8s://cluster/topology', name: 'Cluster Topology',
    description: 'Live nodes, namespaces, and deployments — the shared map both agents reason over.',
    mimeType: 'application/json',
  })
  async topology(uri: string, _ctx: ExecutionContext) {
    const [nodes, deps] = await Promise.all([core.listNode(), apps.listDeploymentForAllNamespaces()]);
    const body = {
      nodes: nodes.items.map((n) => ({ name: n.metadata?.name, unschedulable: n.spec?.unschedulable ?? false })),
      deployments: deps.items.map((d) => ({
        name: d.metadata?.name, namespace: d.metadata?.namespace,
        replicas: d.spec?.replicas, ready: d.status?.readyReplicas })),
    };
    return { contents: [{ uri, mimeType: 'application/json', text: JSON.stringify(body, null, 2) }] };
  }

  @Resource({
    uri: 'k8s://policy/disruption-budgets', name: 'Disruption Budgets & Floors',
    description: 'Live PodDisruptionBudget status plus ops policy min-replica floors — the veto basis.',
    mimeType: 'application/json',
  })
  async budgets(uri: string, _ctx: ExecutionContext) {
    const pdbs = await policy.listPodDisruptionBudgetForAllNamespaces();
    const body = {
      policyFloors: OPS_POLICY.minReplicas,
      pdbs: pdbs.items.map((b) => ({
        name: b.metadata?.name, namespace: b.metadata?.namespace, minAvailable: b.spec?.minAvailable,
        desiredHealthy: b.status?.desiredHealthy, currentHealthy: b.status?.currentHealthy,
        disruptionsAllowed: b.status?.disruptionsAllowed })),
    };
    return { contents: [{ uri, mimeType: 'application/json', text: JSON.stringify(body, null, 2) }] };
  }

  @Resource({
    uri: 'k8s://actions/log', name: 'Remediation Audit Log',
    description: 'Append-only log of every executed and rejected action with rationale.',
    mimeType: 'application/json',
  })
  async actions(uri: string, _ctx: ExecutionContext) {
    return { contents: [{ uri, mimeType: 'application/json', text: JSON.stringify(getActionLog(), null, 2) }] };
  }
}
```

### 3.8 `src/modules/k8s/k8s.prompts.ts` — the 2 prompts

```ts
import { PromptDecorator as Prompt, ExecutionContext } from '@nitrostack/core';
import { getActionLog } from './audit.js';

export class K8sPrompts {
  @Prompt({
    name: 'cluster_health_brief', description: 'Executive cluster health + cost brief for the platform lead.',
    arguments: [{ name: 'lead_name', description: 'Who to address', required: false }],
  })
  async healthBrief(args: { lead_name?: string }, _ctx: ExecutionContext) {
    return [{ role: 'user' as const, content: { type: 'text' as const, text:
`You are the HelmsMan platform analyst. First call get_cluster_state and get_pod_health, then write a
brief for ${args.lead_name ?? 'the Platform Lead'} using these headers:
## Posture (one line: healthy/degraded)
## Cost (total est ₹/hr, and any over-provisioned deployments)
## Risks (pods not Ready, restart storms, cordoned nodes)
## Recommendation (one right-sizing or reliability action)
Use only real tool data. Do not invent numbers.` } }];
  }

  @Prompt({
    name: 'change_record', description: 'Turn the remediation audit log into a formal change record.',
    arguments: [],
  })
  async changeRecord(_args: {}, _ctx: ExecutionContext) {
    return [{ role: 'user' as const, content: { type: 'text' as const, text:
`Produce a change record from this audit log. Headers:
## Executed Changes (action, target, rationale)
## Blocked Changes (action, target, and the exact PDB/policy veto reason — emphasize availability protection)
## Net Effect (replicas/cost before vs after)
Audit log JSON:
${JSON.stringify(getActionLog(), null, 2)}` } }];
  }
}
```

### 3.9 `src/modules/k8s/k8s.module.ts` — module registration

```ts
import { Module } from '@nitrostack/core';
import { K8sTools } from './k8s.tools.js';
import { K8sResources } from './k8s.resources.js';
import { K8sPrompts } from './k8s.prompts.js';

@Module({ name: 'k8s', controllers: [K8sTools, K8sResources, K8sPrompts], providers: [] })
export class K8sModule {}
```

### 3.10 Wire the module into the root app — `src/app.module.ts`

```ts
import { McpApp, Module, ConfigModule } from '@nitrostack/core';
import { K8sModule } from './modules/k8s/k8s.module.js';

@McpApp({ module: AppModule, server: { name: 'helmsman', version: '1.0.0' }, logging: { level: 'info' } })
@Module({ imports: [ConfigModule.forRoot(), K8sModule] })
export class AppModule {}
```

### 3.11 Entry point — `src/index.ts`

```ts
import { McpApplicationFactory } from '@nitrostack/core';
import { AppModule } from './app.module.js';
(async () => { (await McpApplicationFactory.create(AppModule)).start(); })();
```

```
[ ] all 11 files created under src/modules/k8s/ + app.module.ts + index.ts updated
[ ] every relative import ends in .js  (NodeNext ESM requirement)
```

---

## Phase 4 — Verify the safety logic BEFORE the cluster (5 min)

The veto math is the only thing that can silently be wrong. Test it in isolation first.

```bash
npx tsc
node --test dist/modules/k8s/safety.test.js
# expect:  safety self-check: OK
```

If this fails, fix `safety.ts` before going further — nothing else matters until this passes.

```
[ ] tsc compiles with zero errors
[ ] safety self-check: OK
```

---

## Phase 5 — Run the server & connect NitroStudio (20 min)

```bash
npm run dev
# Starts the MCP server over stdio (Studio will spawn it) + widget server on http://localhost:3001
# Leave this running.
```

In **NitroStudio**:

```
[ ] Launch NitroStudio
[ ] Click "Select Project" -> browse to the helmsman/ folder -> "Connect"
      (Studio spawns the MCP subprocess and connects automatically)
[ ] Open Settings -> choose OpenAI or Gemini -> paste your API key -> Save
[ ] Confirm the Tools / Resources / Prompts panels populate with your definitions
```

If the panels are empty: check the terminal running `npm run dev` for a stack trace (usually a
decorator/tsconfig issue or a bad import path — see Phase 8).

---

## Phase 6 — Manually test every primitive in Studio (25 min)

Go panel by panel. This proves the server works before any AI is involved.

**Tools panel:**
```
[ ] get_cluster_state        -> Execute (namespace=shop) -> see checkout-service, replicas 5, estCostInrPerHr
[ ] get_pod_health           -> Execute (namespace=shop) -> 5 pods, ready:true
[ ] check_disruption_safety  -> name=checkout-service, targetReplicas=1
                                -> decision.allowed:false, cites desiredHealthy=3
[ ] check_disruption_safety  -> targetReplicas=3 -> decision.allowed:true
[ ] scale_deployment         -> name=checkout-service, replicas=1, rationale="test"
                                -> ok:false, rejected:true   ***THE MONEY-SHOT — server veto***
[ ] scale_deployment         -> replicas=3, rationale="right-size" -> ok:true, scaledTo:3
[ ] (verify)                 -> in a terminal: kubectl get pods -n shop  (now 3 pods)
```

**Resources panel:**
```
[ ] k8s://cluster/topology            -> live nodes + deployments
[ ] k8s://policy/disruption-budgets   -> PDB with real disruptionsAllowed / desiredHealthy
[ ] k8s://actions/log                 -> shows your REJECTED + EXECUTED entries
```

**Prompts panel:**
```
[ ] cluster_health_brief  -> produces the structured brief from real tool data
[ ] change_record         -> lists the blocked scale-to-1 under "Blocked Changes"
```

**Logs panel:** confirm you see `scale REJECTED` (warn) and `scale EXECUTED` (info) entries.

> Reset between rehearsals: `kubectl scale deploy/checkout-service -n shop --replicas=5`

---

## Phase 7 — Drive the two-agent negotiation (30 min)

The server is neutral; the two agents live in the AI client. Two ways to run the demo:

### 7A. Studio AI chat (single model, dual-persona orchestration) — simplest

In Studio's AI chat, paste this orchestration prompt, then say **"Begin."**:

```
You will simulate a negotiation between two agents that share the HelmsMan MCP tools.

AGENT 1 — FINOPS: goal is to cut cost by eliminating over-provisioning. It calls get_cluster_state,
finds over-provisioned deployments, and PROPOSES a scale_deployment to the lowest replicas it thinks
is justified, quantifying ₹/hr saved.

AGENT 2 — AVAILABILITY GUARDIAN: adversarial gatekeeper. For every proposal it calls
check_disruption_safety and VETOES if decision.allowed is false, citing the PDB desiredHealthy and
disruptionsAllowed. When it vetoes, it tells FinOps the lowest safe replica count.

RULES:
- FinOps must revise upward after any veto (never resubmit a rejected number).
- Only call scale_deployment for the FINAL agreed, safe count.
- Label every line with [FINOPS] or [GUARDIAN] and show each tool call + result.
- End with a one-line summary of replicas before/after and ₹/hr saved.

Begin.
```

Expected trace: FinOps proposes replicas=1 → Guardian vetoes (floor 3) → FinOps revises to 3 →
Guardian approves → `scale_deployment(replicas:3)` executes. Watch `kubectl get pods -n shop` drop 5→3.

### 7B. Two separate agents (Claude Desktop / any MCP client) — most impressive

Because HelmsMan is a standard MCP server, connect it as a tool provider to a client that supports
distinct system prompts, and run FinOps and Guardian as two chats sharing the same server. Use the
verbatim FinOps and Guardian system prompts from the implementation notes. Optional — 7A is enough
to win.

### The kill-shot (needs no AI at all)
Right after the negotiation, in the **Tools panel** manually call
`scale_deployment(name="checkout-service", replicas=1)` → server returns `rejected:true` citing the
**live PDB**. This proves the veto is enforced in code, not by the LLM. Then run the `change_record`
prompt to show it logged under **Blocked Changes**.

```
[ ] Studio AI configured and negotiation runs end-to-end
[ ] kubectl shows pods 5 -> 3 during the demo
[ ] manual scale-to-1 is rejected by the server (kill-shot)
[ ] change_record prompt lists the blocked attempt
```

**Live companion terminal for the projector:**
```bash
kubectl get pods,pdb -n shop -w
```

---

## Phase 8 — Troubleshooting & deployment note

### Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Studio panels empty / server won't boot | decorator/tsconfig | `experimentalDecorators` + `emitDecoratorMetadata` true; `target ES2022`, `module NodeNext` |
| `Cannot find module './x.js'` | ESM needs explicit `.js` | every relative import ends in `.js` even from `.ts` |
| Methods want positional args / `.body` undefined | client-node **0.x vs 1.x** | `npm ls @kubernetes/client-node` → pin `^1.0.0`; this guide is 1.x |
| `ECONNREFUSED` / TLS on every tool | wrong kube context | `kubectl config use-context kind-helmsman` |
| `disruptionsAllowed: null` | PDB status not ready / no ready pods | wait for pods Ready; guard treats null as 0 (fails safe) |
| scale "succeeds" but pods don't change | wrong namespace | confirm `namespace=shop` |
| patch content-type errors | you switched to patch | stay on read-modify-write `replace*` as written |

### Deployment reality (state this in your pitch)
`nitrostack deploy` (via `nitrostack login`) publishes the server to NitroCloud, but HelmsMan needs
line-of-sight to a Kubernetes API server. For the hackathon, **demo on `npm run dev` + NitroStudio
against local `kind`** — this is the deliberate ambition/deploy tradeoff (real cluster control beats
a fake cloud deploy). A production deploy would mount a hosted cluster's kubeconfig as a NitroCloud
secret; out of 24h scope. Say so — it reads as engineering judgment, not a gap.

### Final pre-demo gate
```
[ ] npx tsc                                   -> 0 errors
[ ] node --test dist/modules/k8s/safety.test.js  -> safety self-check: OK
[ ] kubectl get pdb -n shop                   -> disruptionsAllowed is a real number
[ ] Studio: manual scale_deployment(replicas=1) -> rejected:true citing live PDB
[ ] kubectl scale deploy/checkout-service -n shop --replicas=5   (reset for the real run)
```

---

## What was deliberately kept simple (and when to undo it)
- **Cost = requests × replicas** (not live usage) — install `metrics-server` + use `k8s.Metrics` only if a judge asks for real utilization.
- **In-memory audit log** — swap to NitroCloud KV only if audit must survive restarts.
- **Read-modify-write instead of patch** — fine for a single-agent demo; move to server-side apply only under write-conflict races.

Every veto in this build is enforced by a live `PodDisruptionBudget` read from a real cluster — not a
hardcoded label. That is the difference the judge asked for.
