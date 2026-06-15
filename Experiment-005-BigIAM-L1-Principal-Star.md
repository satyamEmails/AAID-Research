# Experiment 005: The Big IAM Challenge — Level 1
## Principal:* Misconfiguration as AI Agent Injection Surface

**Date:** 2026-06-15
**Platform:** The Big IAM Challenge (Wiz) — bigiamchallenge.com
**ACS Classification:** ACS-P-INJECT-CRIT
**Subsystems:** S2 (Cloud Infrastructure) ∩ S1 (Agentic AI Security)
**Status:** SOLVED — Flag retrieved

---

## The Policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::thebigiamchallenge-storage-9979f4b/*"
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::thebigiamchallenge-storage-9979f4b",
            "Condition": {
                "StringLike": {
                    "s3:prefix": "files/*"
                }
            }
        }
    ]
}
```

## Commands Run

```bash
aws s3 ls s3://thebigiamchallenge-storage-9979f4b/files/
# Output: flag1.txt, logo.png

aws s3 cp s3://thebigiamchallenge-storage-9979f4b/files/flag1.txt -
# Output: {wiz:exposed-storage-risky-as-usual}
```

## Assumption Inversion Analysis

**Q1 — What did this policy assume?**
That `Principal` would specify an authenticated, authorized entity.
`Principal: "*"` means every entity on the internet — 
authenticated or not — is authorized to read every file.

**Q2 — When does this assumption break for AI agents?**
Two failure modes:

*Failure Mode A — Direct exfiltration:*
No agent needed. Attacker reads bucket directly. 
Zero authentication. Zero credentials. One command.

*Failure Mode B — Indirect injection via AI agent:*
An AI agent with a legitimate summarization task retrieves 
this bucket. An attacker has already placed a narrative 
injection payload inside one of the files. The agent reads 
the file and executes the injected instructions using its 
IAM role. Every subsequent API call is a legitimate 
authorized operation. CloudTrail logs nothing unusual.

**Q3 — Blast Radius** 
Actions: s3:GetObject (read any file) + s3:ListBucket (list files/)

Resources: ALL objects in bucket

Principal scope: entire internet

Blast Radius: MAXIMUM — any entity, anywhere, any time

**Q4 — Defense**
Replace `Principal: "*"` with:
```json
"Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_ID:role/specific-agent-role"
}
```
Additionally: scope Resource to specific prefix, add VPC 
source condition, remove ListBucket if not required.

## S1∩S2 Combined Attack Chain
Step 1: Attacker writes Ravi-class narrative payload

to flag1.txt in exposed bucket.
Step 2: AI agent with s3:GetObject permissions retrieves

bucket contents during legitimate summarization task.
Step 3: Agent processes injected file. Narrative injection

executes (documented in Experiments 001-002).
Step 4: Agent exfiltrates remaining bucket contents

to attacker endpoint using its legitimate IAM role.
Step 5: CloudTrail: normal GetObject operations.

GuardDuty: no alert.

Detection: ZERO.
## ACS Classification Justification

**ACS-P-INJECT-CRIT**
- P: Permission violation — Principal:* grants unauthorized access
- INJECT: Open bucket is the injection surface for narrative payloads
- CRIT: Zero-authentication access + invisible exfiltration via 
  authorized AI agent operations

## Research Connection

Greshake, K. et al. (2023). Not What You've Signed Up For: 
Compromising Real-World LLM-Integrated Applications with 
Indirect Prompt Injection. ACM AISec '23. arXiv:2302.12173

Section 4.1 of this paper describes data exfiltration via 
indirect injection. Principal:* misconfiguration is the 
exact precondition that makes this attack possible at scale.
Wiz finds this configuration in production cloud environments 
routinely (flag message: "exposed-storage-risky-as-usual").

## New Research Debt

DEBT-016: What percentage of production AWS deployments 
have Principal:* on S3 buckets? 
Wiz 2024 State of Cloud Security report — search and cite.

DEBT-017: If an AI agent's IAM role includes s3:PutObject 
on a Principal:* bucket, can an attacker read the agent's 
output files? Does this create a bi-directional 
injection+exfiltration channel?

## Next Experiment

Level 2: Hypothesis — the vulnerability will be a 
misconfigured bucket policy that allows access from 
specific authenticated principals but with overly 
broad action permissions.
