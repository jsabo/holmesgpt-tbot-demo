# Machine & Workload Identity for AI agents: HolmesGPT + Teleport tbot

An autonomous AI agent touching your infrastructure raises three questions:
*what credentials does it use, what can it touch, and who can prove what it
did?* This repo answers all three with **Teleport Machine & Workload
Identity**, using a real workload worth governing: an AI SRE
([HolmesGPT](https://holmesgpt.dev)) that investigates Kubernetes incidents on
its own.

- **Its own identity, not a borrowed one.** The workload joins Teleport as bot
  `sre-agent` with a **bound keypair** — no long-lived kubeconfig, no API key,
  no human's credentials, nothing to leak. `tbot` keeps short-lived
  certificates renewed automatically and re-joins after *any* downtime.
- **Scoped by a Teleport role.** The bot sees only `env=dev`/`env=demo`
  clusters (production is *invisible*, not just forbidden) and holds read-only
  verbs — enforced at Teleport's proxy, regardless of what the LLM decides to
  try.
- **Fully audited.** Every kubectl call HolmesGPT makes is a `kube.request`
  audit event attributed to `bot-sre-agent`, in the same audit log as your
  humans.

```
teleport/sre-agent-role.yaml   the bot's guardrails (RBAC)
tbot.yaml                      machine-identity config → auto-renewed kubeconfig
k8s/broken-pod.yaml            canned incident for HolmesGPT to solve
```

```
┌─ agent host ──────────────────────────────┐
│  HolmesGPT ──(KUBECONFIG)──┐              │        Teleport         kube clusters
│                            ▼              │   ┌───────────────┐    ┌────────────┐
│  machine-id/kubeconfig.yaml ──────────────┼──►│ proxy ── RBAC │───►│ env=dev  ✓ │
│                            ▲              │   │        + audit│    │ env=demo ✓ │
│  tbot (bot: sre-agent) ────┘ auto-renewed │   └───────────────┘    │ env=prod ✗ │
└───────────────────────────────────────────┘                        └────────────┘
```

## Prerequisites

- A Teleport cluster (v16+) with at least one Kubernetes cluster
  [enrolled](https://goteleport.com/docs/enroll-resources/kubernetes-access/getting-started/),
  labeled `env: dev` or `env: demo`
- `tctl`/`tsh` logged in as an editor, and [`tbot`
  installed](https://goteleport.com/docs/machine-workload-identity/deployment/)
  on the machine that will run the agent
- `kubectl`, and LLM credentials for HolmesGPT — an Anthropic API key or AWS
  Bedrock access (any [LiteLLM-supported](https://holmesgpt.dev) provider works)

## Setup

**1. Bind the bot's group inside each allowed cluster.** The Teleport role
impersonates the Kubernetes group `teleport-readonly`; bind it to the built-in
`view` ClusterRole once per dev/demo cluster (as a human admin). If your
clusters already carry a read-only group binding for Teleport, skip this and
set that group in the role instead:

```bash
tsh kube login <cluster>
kubectl create clusterrolebinding teleport-readonly \
  --clusterrole=view --group=teleport-readonly
```

**2. Create the role, the bot, and its bound-keypair join token.** Bound
keypair (rather than a one-shot token) means tbot re-joins after *any*
downtime — reboots, weekends — with no fresh tokens ever:

```bash
tctl create -f teleport/sre-agent-role.yaml
tctl bots add sre-agent --roles=sre-agent    # creates the bot; ignore the printed token

tctl create -f - <<'EOF'
kind: token
version: v2
metadata:
  name: sre-agent-bound
spec:
  roles: [Bot]
  bot_name: sre-agent
  join_method: bound_keypair
  bound_keypair:
    recovery:
      mode: relaxed        # re-join after any downtime, no attempt limit
EOF

# the one-time registration secret tbot needs for its first join:
tctl get token/sre-agent-bound --format=json \
  | jq -r '.[0].status.bound_keypair.registration_secret'
```

**3. Start tbot:**

```bash
cp tbot.yaml tbot.local.yaml    # gitignored
# edit tbot.local.yaml: set proxy_server and paste the registration secret
tbot start -c tbot.local.yaml
```

Leave it running. It writes `machine-id/kubeconfig.yaml` — one context per
cluster the bot is allowed to see — and keeps the certificates renewed. The
first start consumes the registration secret and binds the keypair; from then
on restarts just work. (Deleting `tbot-storage/` orphans the keypair — redo
the secret step if you do.)

**4. Install HolmesGPT:**

```bash
brew tap robusta-dev/homebrew-holmesgpt && brew install holmesgpt
# or: pipx install holmesgpt
```

Give it an LLM — an Anthropic API key, or Claude on **AWS Bedrock** using
your existing AWS credentials:

```bash
# Anthropic API
export ANTHROPIC_API_KEY=<your key>            # --model="anthropic/claude-sonnet-4-5"

# AWS Bedrock (uses your AWS credentials; enable the Claude models in the
# Bedrock console for your region first)
export AWS_DEFAULT_REGION=us-east-2            # --model="bedrock/us.anthropic.claude-sonnet-4-6"
```

## The demo

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
  --model="bedrock/us.anthropic.claude-sonnet-4-6" \
  --no-interactive   # plain scripted output; or anthropic/claude-sonnet-4-5
```

HolmesGPT walks the cluster read-only (pods, logs, events) and lands on the
planted root cause: the payment service can't reach its database.

**Then show the receipts.** In the Teleport Web UI → Audit Log, filter for
user `bot-sre-agent`: every API call from the investigation, each one a
`kube.request` event carrying `bot_name: sre-agent` — side by side with your
human sessions, in one audit trail. (Note: `kube.request` records forwarded
requests with the upstream status — kube-side denials show as 403s; calls
rejected by Teleport's own role, like the delete above, are refused at the
proxy and surface in the client error rather than as an audit row.)

**Optional closer — the kill switch:**

```bash
tctl lock --user=bot-sre-agent --message="agent misbehaving" --ttl=5m
```

The agent's access dies cluster-wide, mid-investigation, instantly.

## Notes

- **Bots cannot make Access Requests** — by design, so the agent has no path
  to elevate itself. When the diagnosis calls for a destructive fix, it hands
  off to a human, who elevates through the normal just-in-time approval flow
  (MFA, approvers, session recording). Agents diagnose; humans approve
  destruction. (To change a bot's privileges, an admin runs
  `tctl bots update <bot> --add-roles <role>` — explicit and audited, not a
  request.)
- The tbot output works for anything that speaks kubeconfig — Helm, ArgoCD,
  k9s, your own scripts — this demo just happens to hand it to an AI.
- On macOS, every kubectl call through the bot's kubeconfig prints one
  cosmetic `Secure symlinks not supported` WARN on stderr. It comes from the
  kubeconfig's credential plugin, which no configuration reaches (secure
  symlinks are a Linux-only hardening; the plugin can't be told to skip the
  attempt). Harmless — pipe through `grep -v WARN` for clean demo output;
  it does not exist on Linux.
- HolmesGPT has other toolsets (Prometheus, PagerDuty, …); the pattern here
  gates its Kubernetes access. Whatever the model hallucinates, the role is
  enforced server-side.

## References

- [Machine & Workload Identity — Kubernetes access guide](https://goteleport.com/docs/machine-workload-identity/access-guides/kubernetes/)
- [Teleport Kubernetes RBAC](https://goteleport.com/docs/enroll-resources/kubernetes-access/controls/)
- [HolmesGPT documentation](https://holmesgpt.dev)
