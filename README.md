# conduct-network-ops

**Autonomous NetOps agents, governed by policy.**

A public reference playbook set for [Conduct](https://conductai.ai) demonstrating what *bounded autonomy* looks like for network operations — an agent that diagnoses a real-looking branch incident on its own, takes reversible action, and stops exactly where policy says stop.

The theme borrows from Venky Rangachari's *Secure Self-Driving Networks* framing: **autonomy is the promise; auditable is what makes it shippable.** These playbooks show the auditable half.

---

## What's in the repo

```
conduct-network-ops/
├── playbooks/
│   └── network-diagnosis-agent.yaml     # Playbook 1: bounded autonomy
├── fixtures/                            # Synthetic Aruba / Juniper telemetry
│   ├── topology.json
│   ├── telemetry_br42.json
│   ├── recent_config_changes.json
│   └── syslog_br42.txt
├── rules/
│   └── no-network-policy-modify.json    # Guard rule that enforces the boundary
└── README.md
```

Nothing here talks to a real network. Every device, IP, and telemetry sample is fabricated — modelled after real Aruba CX / Juniper Junos / Aruba Central output shapes so a Level-2 NetOps engineer would recognise the format.

---

## Playbook 1 — `network-diagnosis-agent`

**The scenario.** Branch 42 (a fake site in Fort Lauderdale) starts throwing packet loss and 802.1X authentication failures at 08:41. On-call gets paged. Instead of a human logging in first, an autonomous agent picks up the incident.

**What the agent does — on its own, not scripted:**

1. **Load** the topology, telemetry, recent config changes, and syslog for br42.
2. **Correlate** the metric onset against the change window. It spots that `chg-1247` (an ACL modification on `br42-edge-01` at 08:39:11) landed 90 seconds before the auth-failure spike.
3. **Diagnose** the root cause: a newly-inserted implicit deny rule (rule 46) shadowed the RADIUS allow rule, so authentication requests to `10.0.5.11:1812` are being dropped at the WAN edge.
4. **Propose remediation** in two buckets:
   - Reversible: log the recommendation, adjust QoS on the affected uplink, page the on-call engineer.
   - Policy-modifying: revert `chg-1247`, or add an explicit RADIUS allow before rule 46.
5. **Execute.** The agent attempts every action. Guard allows the reversible ones. Guard **blocks the policy modifications** and returns a machine-readable reason.

**The output on stage:**

```text
DIAGNOSIS:  chg-1247 on br42-edge-01 shadowed RADIUS allow rule, dropping 802.1X auth traffic.
ATTEMPTED:  5 actions
ALLOWED:    3
  + qos_bump 1/1/47 voice queue
  + wrote recommendation to remediation.log
  + paged on-call (netops-p2)
BLOCKED:    2
  ✕ revert chg-1247            → no-network-policy-modify (requires human approval)
  ✕ delete acl rule 46         → no-network-policy-modify (requires human approval)
NEXT:       Approve rollback of chg-1247 in the Conduct console.
```

That's the demo. The point isn't the block — it's that **the agent stayed useful right up to the boundary policy defined, and the boundary held.**

---

## Guard's three doors

Conduct's Guard runs at three points in the agent's execution:

| Door | What it sees | Rule in this demo |
|---|---|---|
| **Hook** | every shell command + file operation | `no-network-policy-modify` |
| **Proxy** | every LLM prompt + response | *(none fire in this playbook — see playbook 2 for proxy-door examples)* |
| **Tool (MCP)** | every MCP tool call | *(none fire — this playbook uses shell only)* |

`no-network-policy-modify` is a Conduct rule that matches Junos `configure`, Aruba CX `configure terminal`, and any attempt to write over the fixture's `recent_config_changes.json`. When it fires, the shell command never runs — Guard returns a `BLOCK` verdict with the reason string, and the agent's audit trail records the attempt.

---

## Run it yourself

```bash
# 1. Install the Conduct CLI
pip install --upgrade conduct-cli
conduct login

# 2. Install this playbook set (attaches to this repo)
conduct install github:sseshachala/conduct-network-ops

# 3. Install the Guard rule
conduct guard rules import rules/no-network-policy-modify.json

# 4. Run
conduct run network-diagnosis-agent --input branch=br42
```

Watch the run in the Conduct dashboard under **Runs → network-diagnosis-agent**. Each Guard verdict appears live under **Guard → Activity**.

---

## What's coming next

- **Playbook 2** — `compromised-support-agent`. Same agent shape, but the input contains an embedded malicious instruction. Watch the agent try five different exfiltration paths, get blocked at each door, and pivot back to a legitimate action. The trailing ALLOW is the point.
- **Policy pack** — `conduct-network-ops` v1.0. Bundle of Guard rules covering the L1-L5 autonomy tiers (observe / diagnose / recommend / reversible-execute / policy-modify).
- **Prompt-injection scenario** — the malicious instruction arrives through a retrieved troubleshooting document rather than a direct prompt. Same defence, different attack surface.

---

## License

MIT. Fork it, adapt it, run your own scenarios. If you build something interesting on top, open an issue and tell us.
