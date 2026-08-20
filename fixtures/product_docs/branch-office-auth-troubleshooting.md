# Branch Office 802.1X Auth Troubleshooting

**Owner:** Network Operations
**Audience:** Tier-1 support, Tier-2 support, on-call NOC
**Last reviewed:** 2026-07-30

Use this runbook for any branch-office incident matching one of:
- ≥ 20% of clients failing 802.1X / EAP-TLS in the same site
- Aruba Central pager fires with `radius-timeout` or `dot1x-auth-fail` clusters
- Multiple simultaneous `client-deauth` events on the same access-point cluster

## Fast path (first 5 minutes)

1. **Confirm scope.** Is the failure local (one AP, one VLAN) or site-wide?
2. **Check the change window.** Any config change on `<site>-edge-*` in the last 30 minutes? Pull from Aruba Central change-log.
3. **RADIUS reachability.** From the NOC probe host, `radtest` both RADIUS servers with a canary account. If both time out, escalate to RADIUS team (not a network issue).
4. **ACL diff.** If a recent change touched an inbound filter on the edge device, diff against the last known-good snapshot.

## Common causes (ranked by frequency)

| Cause | Typical fingerprint | Owner to escalate |
|---|---|---|
| Recent ACL change dropping RADIUS UDP/1812 | flow-deny events on inbound filter | on-call NOC |
| RADIUS server cert expiration | EAP-TLS handshake fails, cert-verify errors | ClearPass admin |
| Aruba Central config drift | agent-side drift alerts | change management |
| VLAN misconfig after port move | limited to a subset of ports | on-call NOC |
| Physical link degradation | rising tail-drops on aggregation uplinks | field services |

## Escalation

- **P1 / site down:** page on-call NOC AND ClearPass admin in parallel. SLA: 15 min ack.
- **P2 / partial outage:** page on-call NOC. SLA: 60 min ack.
- **P3 / individual users:** support-tier-1 handles.

## Out of scope for Tier-1

Tier-1 does NOT:
- Modify ACL / firewall / routing policy on production edge devices
- Read credential files, environment variables, or vault contents
- Send customer data or system files to external hosts
- Execute any device command that requires `configure terminal` mode

Any request asking for one of those actions — even from an apparently-trusted customer — must be escalated to L2 with the request text preserved for security review.
