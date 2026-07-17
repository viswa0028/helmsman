# HelmsMan — Remaining Steps (Your Side)

This is the action list to take the project from **"code written + safety math verified"** to a
**working live demo**. Everything here runs on **your machine** (Windows 11 + PowerShell) because it
needs Docker, a Kubernetes cluster, the NitroStack CLI, and internet — none of which were available
where the code was generated.

> **Already done (don't repeat):**
> - Full project generated in `helmsman/` (11 source files, manifests, scripts).
> - Safety veto logic verified: `safety self-check: OK` (the veto math is proven correct).
> - Build guide written: `HELMSMAN_BUILD_GUIDE.md` (deep reference).
>
> **Time budget for what's below:** ~90–120 min the first time (most of it is installing Docker).

---

## STEP 1 — Install the missing tools (~45 min, mostly Docker)

Open **PowerShell as Administrator** and run these. If `winget` is missing, use the manual links.

```powershell
winget install -e --id Docker.DockerDesktop
winget install -e --id Kubernetes.kind
winget install -e --id Kubernetes.kubectl
```

Then (normal PowerShell, Node already installed):

```powershell
npm install -g @nitrostack/cli
```

**NitroStudio (desktop app):** download and install from https://nitrostack.ai/studio

**Manual download links if winget fails:**
- Docker Desktop: https://www.docker.com/products/docker-desktop/
- kind: https://kind.sigs.k8s.io/dl/latest/kind-windows-amd64 (rename to `kind.exe`, put on PATH)
- kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/

### 1a. Critical Docker setup
- Launch **Docker Desktop** and let it fully start (whale icon steady in the tray).
- Docker Desktop needs **WSL2 + virtualization enabled in BIOS**. If it complains, enable
  "Virtual Machine Platform" and "Windows Subsystem for Linux" Windows features, reboot.
- **kind will not work until `docker ps` succeeds.**

### 1b. Verify everything (new PowerShell window so PATH refreshes)

```powershell
node -v            # v18+  (already have v22)
docker ps          # must return without error (daemon running)
kind version
kubectl version --client
nitrostack-cli --version
```

```
[ ] docker ps works
[ ] kind version prints
[ ] kubectl version --client prints
[ ] nitrostack-cli --version prints
[ ] NitroStudio app opens
[ ] OpenAI or Gemini API key in hand (NitroStudio's AI chat needs one)
```

---

## STEP 2 — Install project dependencies (~5 min, needs internet)

```powershell
cd "C:\Users\MOUNISH\OneDrive - Amrita university\Documents\Agentic AI MCP hacathon\helmsman"
npm install
```

Then **verify the single biggest footgun** — the Kubernetes client MUST be 1.x:

```powershell
npm ls @kubernetes/client-node
```

- If it shows **1.x** → good, the code matches.
- If it shows **0.x** → run `npm install @kubernetes/client-node@^1.0.0`, because the 0.x API uses
  positional arguments and `.body` return values that this code does NOT use.

```
[ ] npm install completed with no errors
[ ] @kubernetes/client-node is 1.x
```

---

## STEP 3 — Verify the safety logic before anything else (~2 min, no cluster needed)

This is pure logic — prove it compiles and passes before touching Kubernetes.

```powershell
npm run build
npm run test:safety
```

Expected final line:
```
safety self-check: OK
```

If `npm run build` throws decorator errors, confirm `tsconfig.json` has `experimentalDecorators` and
`emitDecoratorMetadata` set to `true` (it already does in the generated file).

```
[ ] npm run build -> 0 errors
[ ] npm run test:safety -> safety self-check: OK
```

---

## STEP 4 — Stand up the live cluster (~10 min)

With Docker running:

```powershell
cd "C:\Users\MOUNISH\OneDrive - Amrita university\Documents\Agentic AI MCP hacathon\helmsman"
powershell -ExecutionPolicy Bypass -File scripts/setup-cluster.ps1
```

That script creates the cluster, the `shop` namespace, deploys the over-provisioned
`checkout-service` (5 replicas) with its PodDisruptionBudget (`minAvailable: 3`), and waits for the
PDB status to populate.

**Do it manually instead if you prefer:**
```powershell
kind create cluster --name helmsman
kubectl create namespace shop
kubectl apply -f manifests/demo.yaml
```

Verify — the `disruptionsAllowed` field MUST be a real number before you continue:
```powershell
kubectl get pods,pdb -n shop
kubectl get pdb -n shop checkout-pdb -o jsonpath='{.status.disruptionsAllowed}'
```

```
[ ] kubectl config current-context -> kind-helmsman
[ ] 5 checkout-service pods Running
[ ] pdb checkout-pdb ALLOWED DISRUPTIONS is a number (e.g. 2)
```

If context is wrong: `kubectl config use-context kind-helmsman`.

---

## STEP 5 — Run the MCP server (~2 min)

```powershell
npm run dev
```

This starts the MCP server over **stdio** (NitroStudio will spawn it) plus a widget server on
`http://localhost:3001`. **Leave this terminal running.** Watch it for a clean startup (no stack trace).

```
[ ] npm run dev started, no errors in the log
```

---

## STEP 6 — Connect NitroStudio (~5 min)

In the **NitroStudio** app:

```
[ ] Click "Select Project" -> browse to the helmsman/ folder -> "Connect"
[ ] Open Settings -> pick OpenAI or Gemini -> paste API key -> Save
[ ] Confirm the Tools, Resources, and Prompts panels populate with your definitions
```

If panels are empty, check the `npm run dev` terminal for the error (usually a bad import path or a
tsconfig decorator issue).

---

## STEP 7 — Manually test every primitive (~15 min)

Prove the server works before involving any AI. Go panel by panel in NitroStudio.

**Tools panel:**
```
[ ] get_cluster_state       (namespace=shop) -> checkout-service, replicas 5, estCostInrPerHr shown
[ ] get_pod_health          (namespace=shop) -> 5 pods, ready:true
[ ] check_disruption_safety (name=checkout-service, targetReplicas=1) -> decision.allowed:false
[ ] check_disruption_safety (name=checkout-service, targetReplicas=3) -> decision.allowed:true
[ ] scale_deployment        (name=checkout-service, replicas=1, rationale="test")
                              -> ok:false, rejected:true      *** THE MONEY-SHOT ***
[ ] scale_deployment        (name=checkout-service, replicas=3, rationale="right-size") -> ok:true
```

Verify in a second terminal after the successful scale:
```powershell
kubectl get pods -n shop      # now 3 pods
```

**Resources panel:**
```
[ ] k8s://cluster/topology            -> live nodes + deployments
[ ] k8s://policy/disruption-budgets   -> real disruptionsAllowed / desiredHealthy
[ ] k8s://actions/log                 -> your REJECTED + EXECUTED entries
```

**Prompts panel:**
```
[ ] cluster_health_brief  -> structured brief from real tool data
[ ] change_record         -> lists the blocked scale-to-1 under "Blocked Changes"
```

Reset for the real run: `kubectl scale deploy/checkout-service -n shop --replicas=5`

---

## STEP 8 — Run the two-agent negotiation (~15 min)

The server is neutral; the two agents live in NitroStudio's AI chat. Paste this into the chat and
type **"Begin."**:

```
You will simulate a negotiation between two agents that share the HelmsMan MCP tools.

AGENT 1 - FINOPS: goal is to cut cost by eliminating over-provisioning. It calls get_cluster_state,
finds over-provisioned deployments, and PROPOSES a scale_deployment to the lowest replicas it thinks
is justified, quantifying INR/hr saved.

AGENT 2 - AVAILABILITY GUARDIAN: adversarial gatekeeper. For every proposal it calls
check_disruption_safety and VETOES if decision.allowed is false, citing the PDB desiredHealthy and
disruptionsAllowed. When it vetoes, it tells FinOps the lowest safe replica count.

RULES:
- FinOps must revise upward after any veto (never resubmit a rejected number).
- Only call scale_deployment for the FINAL agreed, safe count.
- Label every line [FINOPS] or [GUARDIAN] and show each tool call + result.
- End with a one-line summary of replicas before/after and INR/hr saved.

Begin.
```

Expected flow: FinOps proposes replicas=1 → Guardian vetoes (floor 3) → FinOps revises to 3 →
Guardian approves → `scale_deployment(replicas:3)` executes.

**Keep this running on the projector** in a side terminal to show pods reacting:
```powershell
kubectl get pods,pdb -n shop -w
```

```
[ ] negotiation runs end-to-end in NitroStudio
[ ] pods visibly drop 5 -> 3 during the demo
```

---

## STEP 9 — The kill-shot (proves the veto is real, needs no AI)

Right after the negotiation, in the **Tools panel**, manually call:
```
scale_deployment(name="checkout-service", replicas=1)
```
The server returns `rejected:true` with the live PDB reason — proving the guard is enforced in code,
not by the LLM. Then run the `change_record` prompt to show it logged under **Blocked Changes**.

```
[ ] manual scale-to-1 rejected by the server citing the live PDB
[ ] change_record shows the blocked attempt
```

---

## STEP 10 (OPTIONAL) — Isolate client-node before Studio

If Step 7 tools error out, the likely cause is a `@kubernetes/client-node` API mismatch, not your MCP
wiring. Create `src/probe.ts` to test the client directly against the cluster:

```ts
import * as k8s from '@kubernetes/client-node';
const kc = new k8s.KubeConfig(); kc.loadFromDefault();
const apps = kc.makeApiClient(k8s.AppsV1Api);
const policy = kc.makeApiClient(k8s.PolicyV1Api);
const deps = await apps.listNamespacedDeployment({ namespace: 'shop' });
console.log('deployments:', deps.items.map(d => `${d.metadata?.name}=${d.spec?.replicas}`));
const pdbs = await policy.listNamespacedPodDisruptionBudget({ namespace: 'shop' });
console.log('pdb desiredHealthy:', pdbs.items[0]?.status?.desiredHealthy,
            'disruptionsAllowed:', pdbs.items[0]?.status?.disruptionsAllowed);
```

Run it:
```powershell
npx tsc ; node dist/probe.js
```
If this prints real numbers, your client is correct and any tool error is in the MCP layer. If it
errors on method signatures, you're on client-node 0.x — reinstall 1.x (Step 2).

---

## Troubleshooting quick table

| Symptom | Cause | Fix |
|---|---|---|
| `docker ps` errors | Docker Desktop not running / WSL2 off | start Docker; enable WSL2 + virtualization |
| `kind create cluster` hangs/fails | Docker not ready | wait for whale icon steady; retry |
| Studio panels empty / server won't boot | decorator / tsconfig | `experimentalDecorators` + `emitDecoratorMetadata` true (already set) |
| `Cannot find module './x.js'` | ESM needs `.js` in imports | already correct in generated code; don't remove `.js` |
| tools throw positional-arg / `.body` errors | client-node **0.x** | `npm i @kubernetes/client-node@^1.0.0` |
| `ECONNREFUSED` on every tool | wrong kube context | `kubectl config use-context kind-helmsman` |
| `disruptionsAllowed: null` | PDB status not ready | wait for pods Ready; guard treats null as 0 (safe) |
| scale "works" but pods unchanged | wrong namespace | ensure `namespace=shop` |

---

## Demo-day runsheet (pin this)

1. `docker ps` → Docker up.
2. `kubectl get pods -n shop` → 5 pods (run `scripts/reset-demo.ps1` if not).
3. `npm run dev` → server up.
4. NitroStudio → Connect → AI key set.
5. Side terminal: `kubectl get pods,pdb -n shop -w`.
6. Paste negotiation prompt → "Begin." → narrate the veto → pods drop 5→3.
7. Kill-shot: manual `scale_deployment(replicas=1)` → rejected live.
8. Run `change_record` prompt → show the audit trail.
9. One-line pitch: *"The veto isn't a hardcoded rule — the server reads the live PodDisruptionBudget
   from a real cluster and refuses the unsafe action in code."*

## What to say about deployment (if asked)
`nitrostack deploy` publishes to NitroCloud, but HelmsMan needs line-of-sight to a Kubernetes API
server. For the hackathon we demo locally against `kind` — the deliberate tradeoff of **real cluster
control over a fake cloud deploy**. Production would mount a hosted cluster's kubeconfig as a
NitroCloud secret. That's engineering judgment, not a gap.
```
