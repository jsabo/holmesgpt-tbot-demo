# Governing AI agents with Teleport: identity, RBAC, and audit

AI agents are touching infrastructure two ways today: **autonomous agents**
(an AI SRE investigating alerts on its own) and **interactive agents** (Claude
Code, Cursor, and friends calling tools over MCP on a human's behalf). Both
raise the same three questions: *what credentials does the AI use, what can it
touch, and who can prove what it did?*

This repo demos Teleport answering all three, for both kinds of agent:

- **Part 1 — autonomous:** [HolmesGPT](https://holmesgpt.dev) investigates a
  crashing pod using its **own machine identity** (`tbot`), scoped by a
  Teleport role, every kubectl call audited as `bot-sre-agent`.
- **Part 2 — interactive:** an MCP server enrolled behind Teleport, with
  **per-tool RBAC** (the agent literally cannot see denied tools), just-in-time
  elevation a human approves, and per-tool-call audit.

---

## Part 1 — Autonomous agent: HolmesGPT + Machine ID

- **Its own identity, not a borrowed one.** `tbot` joins Teleport as bot
  `sre-agent` and maintains short-lived certificates — no long-lived
  kubeconfig, no human's credentials, nothing to leak.
- **Scoped by a Teleport role.** The bot sees only `env=dev`/`env=demo`
  clusters (production is *invisible*, not just forbidden) and holds read-only
  verbs — enforced at Teleport's proxy, regardless of what the LLM decides to
  try.
- **Fully audited.** Every kubectl call HolmesGPT makes is a `kube.request`
  audit event attributed to `bot-sre-agent`, in the same audit log as your
  humans.

```
┌─ agent host ──────────────────────────────┐
│  HolmesGPT ──(KUBECONFIG)──┐              │        Teleport         kube clusters
│                            ▼              │   ┌───────────────┐    ┌────────────┐
│  machine-id/kubeconfig.yaml ──────────────┼──►│ proxy ── RBAC │───►│ env=dev  ✓ │
│                            ▲              │   │        + audit│    │ env=demo ✓ │
│  tbot (bot: sre-agent) ────┘ auto-renewed │   └───────────────┘    │ env=prod ✗ │
└───────────────────────────────────────────┘                        └────────────┘
```

### Prerequisites

- A Teleport cluster (v16+) with at least one Kubernetes cluster
  [enrolled](https://goteleport.com/docs/enroll-resources/kubernetes-access/getting-started/),
  labeled `env: dev` or `env: demo`
- `tctl`/`tsh` logged in as an editor, and [`tbot`
  installed](https://goteleport.com/docs/machine-workload-identity/deployment/)
  on the machine that will run the agent
- `kubectl`, and an Anthropic (or other
  [LiteLLM-supported](https://holmesgpt.dev)) API key for HolmesGPT

### Setup

**1. Bind the bot's group inside each allowed cluster.** The Teleport role
impersonates the Kubernetes group `view`; bind it to the built-in `view`
ClusterRole once per dev/demo cluster (as a human admin):

```bash
tsh kube login <cluster>
kubectl create clusterrolebinding teleport-sre-agent-view \
  --clusterrole=view --group=view
```

**2. Create the role and the bot:**

```bash
tctl create -f teleport/sre-agent-role.yaml
tctl bots add sre-agent --roles=sre-agent    # prints a one-time join token
```

**3. Start tbot:**

```bash
cp tbot.yaml tbot.local.yaml    # gitignored
# edit tbot.local.yaml: set proxy_server and paste the join token
tbot start -c tbot.local.yaml
```

Leave it running. It writes `machine-id/kubeconfig.yaml` — one context per
cluster the bot is allowed to see — and keeps the certificates renewed.

**4. Install HolmesGPT:**

```bash
brew tap robusta-dev/homebrew-holmesgpt && brew install holmesgpt
# or: pipx install holmesgpt
export ANTHROPIC_API_KEY=<your key>
```

### The demo

**The bot is an identity, not a credential file:**

```bash
tctl bots ls                # the bot and its role
tctl bots instances ls      # the live, heartbeating instance tbot maintains
```

**Its access is exactly the role, no matter what the AI tries:**

```bash
export KUBECONFIG=$PWD/machine-id/kubeconfig.yaml

kubectl config get-contexts        # dev/demo clusters only — prod isn't even here
kubectl get pods -A                # works: read verbs allowed
kubectl delete pod <any> -n <ns>   # fails: "bot-sre-agent cannot delete resource pods"
```

**Give it an incident and let it investigate** (create the incident as a
human; the bot diagnoses it):

```bash
tsh kube login <cluster> && kubectl apply -f k8s/broken-pod.yaml   # human, own kubeconfig

export KUBECONFIG=$PWD/machine-id/kubeconfig.yaml                  # back to the bot
holmes ask "pods are crashing in namespace demo-incident — find the root cause" \
  --model="anthropic/claude-sonnet-4-5"
```

HolmesGPT walks the cluster read-only (pods, logs, events) and lands on the
planted root cause: the payment service can't reach its database.

**Then show the receipts.** In the Teleport Web UI → Audit Log, filter for
user `bot-sre-agent`: every single API call the AI just made, each one a
`kube.request` event carrying `bot_name: sre-agent` — side by side with your
human sessions, in one audit trail.

**Optional closer — the kill switch:**

```bash
tctl lock --user=bot-sre-agent --message="agent misbehaving" --ttl=5m
```

The agent's access dies cluster-wide, mid-investigation, instantly.

---

## Part 2 — Interactive agents: MCP with per-tool RBAC

The same governance for agents a human drives (Claude Code, Claude Desktop,
Cursor): enroll the MCP server behind Teleport once, and every user's AI gets
one config entry — while Teleport enforces *which tools* each identity may
call and audits every call with its arguments.

### Setup (~10 minutes, reuses an existing Teleport agent)

**1. Enroll the MCP server** — merge `mcp/app-snippet.yaml` into an existing
agent's `teleport.yaml` and restart it. Point `--kubeconfig` at any demo
cluster the agent host can reach.

**2. Create the roles:**

```bash
tctl create -f mcp/roles/mcp-agent-readonly.yaml
tctl create -f mcp/roles/mcp-agent-admin.yaml
tctl users update <demo-user> --set-roles=<existing-roles>,mcp-agent-readonly
```

**3. Connect the AI client** (as the demo user):

```bash
tsh login --proxy=<cluster> --user=<demo-user>
tsh mcp ls                                        # allowed tools per server
tsh mcp config kubernetes-mcp --client-config=claude   # writes the client entry
```

### The demo

1. **RBAC**: list MCP tools in the client — only `*_list` / `*_get` /
   `pods_log` etc. are visible. Ask the AI to *"delete pod X"* — the tool
   isn't even offered; a forced call is denied. Teleport filtered the toolset
   by role, server-side.
2. **Audit**: Web UI → Audit Log → filter `mcp.session.request`. Every tool
   call the agent made is there, arguments included; denied calls show
   `success: false`.
3. **JIT**: `tsh request create --roles mcp-agent-admin --reason "disk full incident"`
   → a human approves in the Web UI (optionally adjusting duration) →
   reconnect the MCP session → **the tool list visibly grows** → the
   delete/scale now succeeds, and is audited.

Note deny rules win: `pods_exec` / `nodes_debug_exec` stay blocked for
`mcp-agent-readonly` holders even while the requested admin role is active —
full access arrives only because the admin role itself carries no deny.

---

## Notes

- **Bots cannot make Access Requests** — by design. When the autonomous agent
  needs a destructive fix, it hands off to a human, who elevates through the
  normal just-in-time approval flow (MFA, approvers, session recording).
  Agents diagnose; humans approve destruction.
- The tbot output works for anything that speaks kubeconfig — Helm, ArgoCD,
  k9s, your own scripts — Part 1 just happens to hand it to an AI.
- HolmesGPT has other toolsets (Prometheus, PagerDuty, …); the pattern here
  gates its Kubernetes access. Whatever the model hallucinates, the role is
  enforced server-side.
- MCP tool names in the roles match kubernetes-mcp-server /
  openshift-mcp-server; verify against `tsh mcp ls` after enrolling.

## References

- [Machine & Workload Identity — Kubernetes access guide](https://goteleport.com/docs/machine-workload-identity/access-guides/kubernetes/)
- [MCP access — tool RBAC](https://goteleport.com/docs/enroll-resources/mcp-access/rbac/)
- [Teleport Kubernetes RBAC](https://goteleport.com/docs/enroll-resources/kubernetes-access/controls/)
- [HolmesGPT documentation](https://holmesgpt.dev)
