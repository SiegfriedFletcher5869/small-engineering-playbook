# A Guide to 3 Python Tenant Subdomain CNAME Checks Before Fintech Apex Migration

Moving a fintech zone off a registrar-specific API makes web routing easier to change, but a successful tenant page is weak evidence that account notices still reach recipients. Resolve that trade-off by treating mail evidence as a release condition. Short answer: use a CNAME for a tenant subdomain that should follow a service hostname, or an A record when you manage a stable IPv4 destination. At the zone apex, use address records for an ordinary standards-based configuration: a CNAME cannot coexist with the required SOA and NS data. Preserve mail records and verify authenticated delivery separately before completing the migration.

The useful question is bigger than which record resolves. A payment notification can depend on records far away from the hostname serving the tenant dashboard. Moving the whole zone changes the administrative boundary around those records, even when the web application itself stays put.

## Should a tenant subdomain use a CNAME record when the apex handles mail?

Yes, if the tenant hostname should follow a service hostname and has no conflicting data. A CNAME says that its owner name is an alias for another name. An A record supplies an IPv4 address; IPv6 uses AAAA. For tenant subdomains, the alias lets the service operator change its destination addresses without asking each customer to edit an address record. The trade-off is dependence on the target name and its resolution. Address records give the zone operator direct control over destinations, along with responsibility for keeping those destinations current. These record semantics come from RFC 1034 and RFC 1035.

Names matter here.

Suppose the illustrative fintech zone is `ledger.example`, the dashboard lives at `portal.ledger.example`, and mail uses `ledger.example` in the visible From header. A CNAME at `portal` need not displace the apex MX records, the SPF TXT record, the DKIM selector records, or the DMARC policy under `_dmarc`. A CNAME at the apex runs into the coexistence rule before any mail experiment can make it acceptable. A separately delegated tenant zone has its own apex, so counting labels is not enough to detect this boundary.

DMARC adds another distinction. Under RFC 7489, a passing result requires an authenticated SPF or DKIM identity to align with the domain in the message's From header. Resolving the dashboard says nothing about that alignment. Nor does an SMTP acceptance response establish inbox placement. For this migration, keep routing correctness, authentication results, and observed receipt as separate pieces of evidence.

The data flow can stay small: export the source zone into a provider-independent representation, build the proposed destination snapshot, compare the records that support the actual sending paths, and attach controlled receipt observations plus reviewed authentication results to the release decision. Python can turn that decision into a repeatable gate. The destination API belongs behind an adapter; the evidence should survive replacing that adapter.

## Three checks before changing authority

This runnable example uses only the Python standard library. All names and observations are illustrative fixtures, including the two received messages. It performs no DNS lookup and sends no mail. Its three checks cover ordinary record conflicts, preservation of declared mail dependencies, and a deliberately small evidence policy.

```python
from copy import deepcopy
from ipaddress import IPv4Address

APEX = "ledger.example"
MAIL_NAMES = {
    APEX,
    "_dmarc.ledger.example",
    "notice._domainkey.ledger.example",
    "bounce.ledger.example",
}

source = {
    APEX: {
        ("SOA", "source authority metadata"),
        ("NS", "ns.source.example."),
        ("A", "192.0.2.10"),
        ("MX", "10 inbound.example."),
        ("TXT", "v=spf1 -all"),
    },
    "portal.ledger.example": {("CNAME", "edge.service.example.")},
    "_dmarc.ledger.example": {("TXT", "v=DMARC1; p=none")},
    "notice._domainkey.ledger.example": {
        ("CNAME", "notice.keys.example."),
    },
    "bounce.ledger.example": {("TXT", "v=spf1 -all")},
}
candidate = deepcopy(source)
candidate[APEX].difference_update(
    {record for record in candidate[APEX] if record[0] in {"SOA", "NS"}}
)
candidate[APEX].update({
    ("SOA", "destination authority metadata"),
    ("NS", "ns.destination.example."),
})

def routing_ok(zone):
    for name, records in zone.items():
        kinds = {kind for kind, _ in records}
        aliases = [value for kind, value in records if kind == "CNAME"]
        if aliases and (name == APEX or kinds != {"CNAME"} or len(aliases) != 1):
            return False
        for kind, value in records:
            if kind == "A":
                IPv4Address(value)
    return True

def mail_snapshot(zone):
    return {
        name: {record for record in zone.get(name, set())
               if record[0] not in {"SOA", "NS", "A", "AAAA"}}
        for name in MAIL_NAMES
    }

# Synthetic results, standing in for independently collected evidence.
observations = [
    {"flow": "account_notice", "received": True, "dmarc_pass": True},
    {"flow": "payment_notice", "received": True, "dmarc_pass": True},
]
required_flows = {"account_notice", "payment_notice"}
evidence_ok = (
    {row["flow"] for row in observations} == required_flows
    and all(row["received"] and row["dmarc_pass"] for row in observations)
)
checks = {
    "routing": routing_ok(candidate),
    "mail_preserved": mail_snapshot(source) == mail_snapshot(candidate),
    "delivery_evidence": evidence_ok,
}
print(checks)
if not all(checks.values()):
    raise SystemExit("Hold the migration for review")
```

The strings labeled authority metadata are placeholders in an internal snapshot, not wire-format SOA records. The SPF values deliberately authorize no senders; they make the fixture inert, not a production sending configuration. Replace the fixture with normalized exports and actual evidence before using the gate for a release. An invalid IPv4 value raises an exception, which also prevents approval.

Keep the boundary explicit: this checks a declared subset of ordinary DNS data. It is not a zone-file parser, DNSSEC validator, DMARC evaluator, or proof that every sending path has been inventoried. The CNAME conflict check deliberately excludes DNSSEC-related record types from its input model; RFC 2181 describes the relevant coexistence exception. A production adapter must handle that data deliberately rather than treating this small model as a complete DNS implementation.

## Why preservation and receipt are different gates

Exact preservation is useful during an API migration because it narrows what changed. If the destination export differs on a DKIM selector target or an SPF policy, stop and explain the difference before moving authority. Preserve the old values while changing the management interface. Updating authentication policy at the same time makes a failed receipt harder to interpret.

But equality has limits.

The example's mail dependency list is explicit, and that is both its strength and its trap. It can miss an unlisted selector, a separate return-path domain, or a sending subdomain with its own policy. Inventory those names from the configured sending paths before taking the baseline. The model also omits TTLs and DNSSEC state, which need their own migration checks. Record text that looks different may be equivalent after parsing, such as a TXT value split into multiple character strings. Compare normalized data, retaining the original export for review, rather than silently rewriting policies to make a diff disappear.

For deliverability evidence, exercise the notices that matter to this fintech workflow. Capture the message identity, sending path, observation time, recipient-side authentication results, and whether the controlled recipient actually received it. Do not log the notification body merely to prove routing worked. The two flows in the example are a proposed acceptance policy for the illustration, not a measured benchmark or a universal sample size. Broader sending patterns require broader coverage.

DMARC aggregate reports can complement these observations with authentication and alignment results across reported traffic. RFC 7489 describes that reporting mechanism. Reports are not an instantaneous transaction receipt, and their coverage cannot establish inbox placement for every recipient. If reports have not yet covered a sending path, record that evidence gap and decide when it can be reviewed; an empty report set should not become an automatic pass. A preserved `p=none` policy is also not evidence that mail is authenticated. It describes a requested policy.

This distinction is useful for an eval-driven AI application team. A deterministic harness can compare snapshots and assert required evidence fields without spending prompt tokens on every DNS row. If an assistant summarizes a migration diff, keep that summary advisory and give it the smallest relevant evidence set. The gate should remain inspectable Python with named failure reasons. Moving from a notebook to production then means replacing fixture inputs, storing evidence, and enforcing the same assertions in the deployment path.

## Finish the cutover with an evidence trail

Before changing delegation, confirm that the destination authoritative servers return the intended zone data and that the required tenant routing and mail names exist there. Check the actual zone apex, the planned address ownership, TTLs, and any DNSSEC delegation requirements. RFC 1034 explains why cached data means a delegation change is not observed everywhere at once; a local successful query is a useful observation, not a global completion signal.

Keep the source zone available during the transition. Record the prior delegation and the conditions for reverting it, remembering that reversal also interacts with caches. If the destination adapter times out during a write, reconcile the observed records before retrying: an uncertain API response does not tell you whether the write happened. Make operations repeatable, attach the snapshot identifier to each deployment, and expose a failed record comparison or missing receipt as a specific reason to hold activation.

Finally, review controlled receipts and authentication evidence against the baseline, including the sending paths covered and the observations still missing. Store evidence identifiers and timestamps with the migration decision so another engineer can inspect it without reconstructing a notebook session. Complete the registrar API move when the intended routing works, the mail dependencies are accounted for, and the agreed delivery observations pass. That is the practical decision rule for this job: choose the record that fits the name, then require evidence for the messages customers depend on.

## References

The DNS specifications below establish alias behavior, address records, zone authority, and caching. The mail specifications separate SPF and DKIM authentication from DMARC alignment and reporting.

## Sources

- RFC 1034, Domain Names Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, Domain Names Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 2181, Clarifications to the DNS Specification: https://datatracker.ietf.org/doc/html/rfc2181
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
