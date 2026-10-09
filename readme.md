
---

# AWS CSPM in Action: From Security Findings to Verified Remediation

## Project Overview

This project demonstrates a practical Cloud Security Posture Management (CSPM) workflow using **Prowler Local Server** to assess AWS security posture, identify misconfigurations, prioritize risks, investigate root causes, remediate security issues, and validate the results through re-scanning.

Unlike a basic CSPM scan, this project focuses on the complete security lifecycle:

```text
Detect
  ↓
Prioritize
  ↓
Investigate
  ↓
Remediate
  ↓
Verify
```

The project intentionally introduced two controlled AWS security misconfigurations and used Prowler to detect and validate their remediation.

---

## Real-World Scenario

Imagine an AWS environment where a security team receives findings from a CSPM platform.

The environment contains multiple security findings, but not every finding represents a separate security problem. A single misconfiguration can violate multiple security controls and generate several findings.

The security analyst must therefore:

1. Identify the underlying root cause.
2. Assess the actual exposure and potential impact.
3. Prioritize the most important security risks.
4. Remediate the root cause rather than individual findings.
5. Re-scan the environment to verify that the security issue has been resolved.

This project simulates that workflow using a controlled AWS environment.

---

## Project Objectives

The main objectives of this project are:

- Understand how CSPM identifies AWS security misconfigurations.
- Learn how to consolidate multiple findings into their underlying root causes.
- Practice risk prioritization based on exposure and potential impact.
- Investigate security findings using AWS Console and Prowler evidence.
- Remediate AWS security misconfigurations.
- Validate remediation through a follow-up Prowler scan.
- Understand the difference between **finding count** and **actual security issues**.

---

## Security Scenarios

Two intentional misconfigurations were introduced for controlled testing.

### Scenario 1 — Internet-Exposed SSH

An EC2 instance was associated with a dedicated Security Group containing:

```text
Inbound TCP 22
Source: 0.0.0.0/0
```

This configuration exposes the SSH management port to the Internet.

The objective was to demonstrate how a single Security Group rule can result in multiple CSPM findings related to network exposure.

---

### Scenario 2 — S3 Public Access Protection Disabled

A dedicated S3 bucket was created with:

```text
Block Public Access: Disabled
```

No public bucket policy or public object access was configured.

The objective was to demonstrate how CSPM evaluates storage security controls and identifies weakened public-access protection.

---

## CSPM Workflow

The project followed this workflow:

```text
AWS Environment
      ↓
Baseline Assessment
      ↓
Intentional Misconfiguration
      ↓
Prowler Scan
      ↓
Security Findings
      ↓
Risk Prioritization
      ↓
Root Cause Investigation
      ↓
Remediation
      ↓
Prowler Re-scan
      ↓
Verification
```

---

## Tools & Technologies

| Category | Technology |
|---|---|
| CSPM | Prowler Local Server |
| Cloud Platform | AWS |
| Container Platform | Docker Desktop |
| CLI | AWS CLI |
| Shell | PowerShell |
| Configuration | Docker Compose |
| IAM | IAM Role + AssumeRole |

---

Untuk GitHub, saya sarankan bagian ini dibuat **langsung dalam bentuk clean implementation path** yang mengikuti setup final kita. Bagian error/troubleshooting kita pisahkan nanti.

Berikut draft yang bisa langsung masuk README.

---

# Prerequisites

Before starting this project, ensure the following are available:

- AWS account with permission to create and manage the required IAM and lab resources.
- Docker Desktop with Docker Compose.
- AWS CLI installed and configured.
- PowerShell.
- Basic understanding of IAM, EC2, Security Groups, and Amazon S3.
- A dedicated local directory for the Prowler Local Server configuration.

Verify the AWS CLI and Docker installation:

```powershell
aws --version
docker --version
docker compose version
```

![](image//Screenshot%202026-10-05%20213307.png)
----
Verify that the AWS CLI can authenticate successfully:

```powershell
aws sts get-caller-identity
```

The command should return the current AWS account and identity information.

---

# Phase 0 — Prowler Local Server Setup

## 0.1 Create the Project Directory

Create a dedicated directory for the Prowler Local Server:

```powershell
mkdir prowler-local
cd prowler-local
```

Verify the current directory:

```powershell
pwd
```

---

## 0.2 Download the Latest Prowler Local Server Configuration

Prowler officially provides a Docker Compose installation for Windows PowerShell. The recommended approach is to retrieve the latest release version and download the corresponding `docker-compose.yml` and `.env` files. [GitHub](https://github.com/prowler-cloud/prowler?utm_source=chatgpt.com)

```powershell
$VERSION = (Invoke-RestMethod -Uri "https://api.github.com/repos/prowler-cloud/prowler/releases/latest").tag_name

Invoke-WebRequest -Uri "https://raw.githubusercontent.com/prowler-cloud/prowler/refs/tags/$VERSION/docker-compose.yml" -OutFile "docker-compose.yml"

Invoke-WebRequest -Uri "https://raw.githubusercontent.com/prowler-cloud/prowler/refs/tags/$VERSION/.env" -OutFile ".env"
```

Verify the downloaded files:

```powershell
Get-ChildItem
```

Expected files:

```text
docker-compose.yml
.env
```

---
![](image/Screenshot%202026-10-08%20182427.png)
---

## 0.3 Start Prowler Local Server

Start the Prowler services:

```powershell
docker compose up -d
```

Verify the containers:

```powershell
docker compose ps
```

The main services should be running/healthy.
![](image/Screenshot%202026-10-06%20073017.png)
---

Prowler Local Server is then accessible through:

```text
http://localhost:3000
```
![](image/Screenshot%202026-10-06%20074458.png)
---

Prowler's current documentation uses this address for the Local Server web UI. [GitHub](https://github.com/prowler-cloud/prowler?utm_source=chatgpt.com)

Create a local Prowler account using the Web UI.

---

## 0.4 Configure AWS Credential Access for Docker

When using **Connect assuming IAM Role** with Prowler Local Server in Docker, the containers may need access to the host's AWS configuration and credentials. Prowler specifically documents mounting the local `.aws` directory into the `api`, `worker`, and `worker-beat` services. [GitHub](https://github.com/prowler-cloud/prowler/blob/master/docs/troubleshooting.mdx?utm_source=chatgpt.com)

To keep the official `docker-compose.yml` unchanged, create:

```text
docker-compose.override.yml
```

with:

```yaml
services:
  api:
    volumes:
      - "${USERPROFILE}/.aws:/home/prowler/.aws:ro"

  worker:
    volumes:
      - "${USERPROFILE}/.aws:/home/prowler/.aws:ro"

  worker-beat:
    volumes:
      - "${USERPROFILE}/.aws:/home/prowler/.aws:ro"
```

The `:ro` flag mounts the directory as **read-only**.

Validate the merged Compose configuration:

```powershell
docker compose config
```

Then apply the configuration:

```powershell
docker compose up -d
```

---

# AWS IAM Configuration

## 1. Create the Prowler Scan Role

Prowler Local Server can connect to AWS using an **IAM Role**. For a single-account lab, create the role using the Prowler-provided CloudFormation template. Prowler documents IAM Role-based access as the recommended approach for this workflow. [GitHub](https://github.com/prowler-cloud/prowler/blob/master/docs/troubleshooting.mdx?utm_source=chatgpt.com)

Use the following configuration:

| Parameter | Value |
|---|---|
| `DeployLocalRole` | `true` |
| `DeployStackSet` | `false` |
| `EnableOrganizations` | `false` |
| `EnableS3Integration` | `false` |
| `AccountId` | `<AWS_ACCOUNT_ID>` |
| `IAMPrincipal` | `<IAM_USER_OR_PRINCIPAL>` |
| `ExternalId` | `<EXTERNAL_ID_FROM_PROWLER>` |

For a single-account assessment, `DeployStackSet` and Organizations-related options are not required.

`EnableS3Integration` should remain disabled unless S3 report-storage integration is specifically required.

---
![](image/Screenshot%202026-10-06%20193927.png)
---

## 2. IAM Permissions

The Prowler scan role should provide **read-only security assessment access**.

The Prowler-provided role configuration uses permissions such as:

```text
SecurityAudit
ViewOnlyAccess
```

The purpose is to allow Prowler to inspect AWS configuration without giving the scanning role administrative permissions. Prowler documents this read-only role model for AWS assessment. [GitHub](https://github.com/prowler-cloud/prowler?utm_source=chatgpt.com)

---

## 3. Configure AWS CLI Profile

The local AWS CLI must have a valid source credential.

Verify:

```powershell
aws sts get-caller-identity
```

For the Local Server Docker workflow, the `.aws` directory is mounted into the required containers so that Prowler can access the AWS configuration and credentials. Prowler documents this specifically for IAM-role connections in Docker. [GitHub](https://github.com/prowler-cloud/prowler/blob/master/docs/troubleshooting.mdx?utm_source=chatgpt.com)

The Local Server environment uses:

```text
AWS_PROFILE=default
```

The important distinction is:

```text
AWS CLI profile
        ↓
Source AWS credentials
        ↓
AssumeRole
        ↓
ProwlerScan
```

The profile itself is **not an IAM identity**; it is a local AWS CLI configuration that determines which credentials/configuration are used.

---
![](image/Screenshot%202026-10-06%20194936.png)
---

## 4. Configure Prowler AWS Provider

In Prowler Local Server:

```text
Add Provider
    ↓
Amazon Web Services
    ↓
IAM Role
```

Enter:

```text
Role ARN:
arn:aws:iam::<AWS_ACCOUNT_ID>:role/ProwlerScan
```

Use the **External ID generated by the Local Server**.

Then select the appropriate scan behavior and connect the provider.

After a successful connection, the provider should report:

```text
Provider Connected
```

At this point, the Prowler Local Server is ready to perform AWS security assessments.

---
![](image/Screenshot%202026-10-08%20184210.png)
---

## Phase 1 — Baseline Assessment

### 1.1 Run the Baseline Scan

Start a Prowler scan against the connected AWS provider.

```text
Prowler Local Server
        ↓
AWS Provider
        ↓
Baseline Scan
        ↓
Security Findings
```

The baseline scan is performed before introducing any intentional security misconfigurations.

---
![](image/Screenshot%202026-10-08%20213026.png)
---

### 1.2 Review Security Posture

Review the main Overview dashboard and record:

- Threat Score
- Total Findings
- Passed checks
- Failed checks
- Severity distribution
- Attack surface indicators

The purpose is to understand the existing security posture before the controlled experiment.

---
![](image/Screenshot%202026-10-07%20081155.png)
---

### 1.3 Review Existing Findings

Open the **Findings** page and review the existing failed checks.

At this stage:

> **Do not remediate existing findings unless they are directly relevant to the controlled experiment.**

The goal is to distinguish:

```text
Existing Findings
        vs.
Intentional Project 2 Findings
```

This distinction becomes important later when validating our remediation.

---
![](image/Screenshot%202026-10-07%20181558.png)
---
### 1.4 Understand Finding Context

For selected findings, review:

```text
Finding
   ↓
Severity
   ↓
Affected Resource
   ↓
Evidence
   ↓
Recommendation
```

This establishes the investigation workflow that will be used in later phases.

---
![](image/Screenshot%202026-10-07%20183611.png)
---

## Phase 2 — Controlled Misconfigurations

### Objective

Introduce two controlled AWS security misconfigurations to evaluate Prowler's detection capabilities:

1. An EC2 Security Group allowing SSH access from the Internet.
2. An S3 bucket with Block Public Access disabled.

These configurations were created specifically for testing in a controlled lab environment. The objective was to observe how Prowler identifies security control failures before investigating and remediating their root causes.

### 2.1 — Misconfiguration #1: Internet-Exposed SSH

An EC2 instance named `Test-EC2-CSPM` was created and associated with a dedicated Security Group named `Test-SG-CSPM`.

The Security Group was intentionally configured with the following inbound rule:

| Setting | Value |
|---|---|
| Protocol | TCP |
| Port | 22 (SSH) |
| Source | `0.0.0.0/0` |
| Purpose | Controlled security testing |

![](image/Screenshot%202026-10-07%20184818.png)
---
This configuration permits connection attempts to the SSH port from any IPv4 address.

**Security risk:** Exposing a management port to the Internet increases the attack surface and may expose the instance to brute-force attempts, credential attacks, and other remote access threats.

**Expected result:** Prowler should identify failed security checks related to unrestricted SSH ingress and EC2 network exposure.

### 2.2 — Misconfiguration #2: S3 Block Public Access Disabled

A dedicated S3 bucket named `bucket-test-cspm` was created for this experiment.

The bucket-level **Block Public Access** settings were intentionally disabled to test Prowler's ability to identify weakened public-access protections.

| Setting | Value |
|---|---|
| Resource | Dedicated S3 lab bucket |
| Block Public Access | Disabled |
| Bucket policy granting public access | Not configured |
| Public ACLs | Not configured |
| Test data | No sensitive data |

![](image/Screenshot%202026-10-07%20185534.png)
---

**Security risk:** Disabling Block Public Access removes an important protective control and increases the risk of unintended public exposure if another permission or access configuration is misconfigured.

**Expected result:** Prowler should identify a failed check related to the bucket's public-access protection.

> **Safety note:** The bucket was not intentionally made publicly accessible. No public bucket policy or public ACL was added. The experiment tested a weakened security control, not the exposure of actual data.

### 2.3 — Run Prowler Assessment

After both misconfigurations were introduced, a new assessment was initiated using Prowler Local Server.

The purpose was to detect both configurations in a single scan.

The scan results contained four failed checks associated with the two intentional misconfigurations:

| Root cause | Related failed checks |
|---|---:|
| EC2 Security Group / SSH exposure | 3 |
| S3 Block Public Access disabled | 1 |
| **Total** | **4** |

![](image/Screenshot%202026-10-08%20090646.png)
---
The EC2-related checks evaluated the security configuration from multiple perspectives, including Security Group ingress and EC2 network exposure.

### 2.4 — Detection Results

The scan demonstrated that a single underlying misconfiguration can generate multiple CSPM findings.

The resulting findings were grouped into two root causes rather than treated as four unrelated security issues.

This established the starting point for the next phase:

- Prioritize the actual security risks.
- Investigate the affected resources.
- Identify the underlying causes.
- Determine the appropriate remediation.

![](image/Screenshot%202026-10-08%20091454.png)
---

### Key Observation

**Finding count does not equal misconfiguration count.**

A single insecure configuration may violate multiple security controls. Effective CSPM operations require consolidating related findings around their underlying root causes rather than treating every failed check as a separate problem.

---

## Phase 3 — Risk Prioritization & Investigation

### Objective

Analyze the security findings generated by Prowler, consolidate related findings into their underlying root causes, and prioritize remediation based on exposure, potential impact, and resource context.

The objective was to move beyond the finding count and understand which configurations presented the most significant security risks.

### 3.1 — Consolidate Related Findings

The assessment identified four failed checks associated with the two intentional misconfigurations.

| Root Cause | Related Findings | Affected Resource |
|---|---:|---|
| Unrestricted SSH ingress | 3 | EC2 instance and Security Group |
| S3 Block Public Access disabled | 1 | S3 bucket |
| **Total** | **4** | **2 underlying issues** |

The three EC2-related findings evaluated different aspects of the same underlying network exposure. They were therefore grouped into one root cause rather than treated as three independent security problems.

This consolidation helped avoid duplicating remediation efforts.

### 3.2 — Prioritize Security Risks

The two root causes were assessed using the following considerations:

- **Exposure:** Whether an external actor could reach the affected service or resource.
- **Potential impact:** The consequences if the weakness were exploited or combined with another misconfiguration.
- **Resource context:** The purpose of the resource and its current access configuration.
- **Existing safeguards:** Whether other security controls limited the practical exposure.

Based on the lab environment, the following remediation order was selected.

| Priority | Root Cause | Risk Assessment |
|---|---|---|
| P1 | EC2 SSH exposed to the Internet | TCP port 22 was reachable from any IPv4 address, exposing a management service to external connection attempts. |
| P2 | S3 Block Public Access disabled | A preventive security control was weakened. However, no public bucket policy or public ACL was configured, and the lab bucket contained no sensitive data. |

These priorities represent the analyst's contextual assessment for this lab, rather than a direct ranking based solely on Prowler's severity labels.

### 3.3 — Investigate EC2 SSH Exposure

The EC2-related findings were investigated in both Prowler and the AWS Console.

The investigation confirmed:

- The affected instance was `Test-EC2-CSPM`.
- The instance was associated with `Test-SG-CSPM`.
- The Security Group allowed inbound TCP port 22 from `0.0.0.0/0`.

![](image/Screenshot%202026-10-08%20091454.png)
---

**Root cause:** An unrestricted inbound SSH rule exposed the instance's management port to connection attempts from the Internet.

**Security impact:** Internet exposure increases the attack surface and can expose SSH to brute-force attempts, credential attacks, or exploitation of weaknesses in the remote access configuration.

The remediation decision was to remove the unrestricted SSH inbound rule. SSH access would not be retained from the Internet for this lab.

### 3.4 — Investigate S3 Public Access Protection

The S3 finding was investigated by reviewing the bucket's public-access settings and access configuration.

The investigation confirmed:

- The affected resource was `bucket-test-cspm`.
- Bucket-level Block Public Access was disabled.
- No public bucket policy or public ACL was configured.
- The bucket contained no sensitive test data.

![](image/Screenshot%202026-10-09%20074446.png)
---

**Root cause:** Bucket-level Block Public Access protection had been intentionally disabled.

**Security impact:** Disabling these safeguards weakens an important preventive control and increases the possibility of unintended public exposure if another access configuration is introduced incorrectly.

The finding indicated a weakened security control; it did not, by itself, prove that the bucket's contents were publicly accessible.

The remediation decision was to re-enable the bucket-level Block Public Access settings.

### 3.5 — Investigation Summary

The investigation established a clear relationship between the findings and the underlying AWS configurations:

| Finding Group | Verified Root Cause | Decision |
|---|---|---|
| EC2-related checks | Unrestricted SSH ingress | Prioritize first |
| S3-related check | Block Public Access disabled | Remediate second |

The investigation was completed before any configuration changes were made.

The two root causes were then remediated in the next phase, followed by a new Prowler assessment to validate the results.

### Key Lesson

**Finding count does not equal risk, and severity alone does not determine remediation priority.**

Effective CSPM investigation requires consolidating related findings, validating the affected resources, understanding actual exposure, and prioritizing remediation based on the underlying security risk.

---

## Phase 4 — Remediation

### Objective

Remediate the two confirmed AWS security misconfigurations identified during the investigation phase.

The remediation focused on correcting the underlying causes rather than addressing each failed Prowler check individually.

### 4.1 — Remediate EC2 SSH Exposure

**Root cause:** The Security Group `Test-SG-CSPM` allowed inbound SSH traffic from any IPv4 address.

Before remediation:

| Setting | Value |
|---|---|
| Protocol | TCP |
| Port | 22 (SSH) |
| Source | `0.0.0.0/0` |

![](image/Screenshot%202026-10-08%20092808.png)
---

**Remediation action:** Removed the inbound SSH rule from `Test-SG-CSPM`.

The EC2 instance and Security Group were retained. Only the unnecessary inbound SSH rule was removed.

**Reasoning:** The lab did not require SSH access from the Internet. Removing the rule eliminated the unrestricted inbound SSH exposure without deleting the associated infrastructure.

In a production environment, administrative access should be provided through an approved mechanism, such as AWS Systems Manager Session Manager or a suitably restricted management path, according to operational requirements.

### 4.2 — Restore S3 Public Access Protection

**Root cause:** Bucket-level Block Public Access was disabled on `bucket-test-cspm`.

Before remediation:

| Setting | Value |
|---|---|
| Resource | `bucket-test-cspm` |
| Block Public Access | Disabled |
| Public bucket policy | Not configured |
| Public ACLs | Not configured |

![](image/Screenshot%202026-10-08%20092843.png)
---

**Remediation action:** Re-enabled the bucket-level Block Public Access settings.

The bucket was retained, and no public bucket policy or public ACL was introduced.

**Reasoning:** Block Public Access provides an important preventive control against unintended public exposure. Restoring the protection reduced the risk associated with future access-configuration mistakes.

The remediation targeted the bucket-level configuration used in the controlled experiment, rather than changing unrelated account-level settings.

### 4.3 — Remediation Summary

| Root Cause | Remediation | Intended Outcome |
|---|---|---|
| Unrestricted SSH ingress | Removed inbound TCP port 22 rule | Eliminate the unnecessary Internet-facing SSH rule |
| S3 public-access protection disabled | Re-enabled bucket-level Block Public Access | Restore the preventive public-access control |

Both changes addressed the underlying configuration weaknesses identified during the investigation.

### Important Note

Remediation does not end when a configuration is changed. A follow-up CSPM assessment is required to determine whether the relevant security checks recognize the corrected configuration.

The next phase validates both changes through a new Prowler scan.

### Key Lesson

**Remediate the root cause, not each finding individually.**

A single Security Group change can address several related failed checks. Correcting the underlying configuration is more effective than treating every finding as a separate remediation task.

---

## Phase 5 — Validation

### Objective

Validate the effectiveness of the remediation actions by running a new Prowler assessment against the AWS environment.

The objective was to confirm that the two intentional security misconfigurations had been corrected and that the corresponding Prowler checks recognized the updated configurations.

### 5.1 — Run a Follow-up Assessment

After completing both remediation actions, a new scan was initiated using Prowler Local Server.

The follow-up assessment evaluated the updated AWS configuration and provided new results for comparison with the initial findings.

The validation focused specifically on the security checks associated with the two intentional misconfigurations.

### 5.2 — Verify EC2 Security Group Remediation

The original configuration allowed inbound SSH traffic from any IPv4 address:

| Setting | Before | After |
|---|---|---|
| Protocol | TCP | TCP |
| Port | 22 (SSH) | No inbound SSH rule |
| Source | `0.0.0.0/0` | Rule removed |

The unrestricted SSH ingress rule was removed from `Test-SG-CSPM`.

Following the new assessment, the related checks covering Security Group ingress and EC2 SSH exposure were identified as **PASS**.

![](image/Screenshot%202026-10-09%20080107.png)
---

**Validation result:** The intended remediation was recognized by Prowler.

### 5.3 — Verify S3 Public Access Protection

The original S3 bucket configuration had Block Public Access disabled.

| Setting | Before | After |
|---|---|---|
| Resource | `bucket-test-cspm` | `bucket-test-cspm` |
| Block Public Access | Disabled | Enabled |
| Public bucket policy | Not configured | Not configured |
| Public ACLs | Not configured | Not configured |

Bucket-level Block Public Access was re-enabled.

Following the new assessment, the corresponding Prowler check was identified as **PASS**.

![](image/Screenshot%202026-10-09%20080001.png)
---

**Validation result:** Prowler recognized the restored bucket-level public-access protection.

### 5.4 — Before-and-After Results

The four failed checks associated with the two intentional misconfigurations were validated after remediation.

| Root Cause | Related Checks | Before | After |
|---|---:|---|---|
| EC2 SSH exposure | 3 | FAIL | PASS |
| S3 Block Public Access disabled | 1 | FAIL | PASS |
| **Total** | **4** | **4 failed checks** | **4 passed checks** |

This result confirms that the targeted security controls were corrected and recognized by the follow-up assessment.

The result does not mean that every finding in the AWS account was resolved. Existing findings unrelated to this controlled experiment remained outside the remediation scope.

### 5.5 — Validation Summary

The validation phase completed the project's security workflow:

```text
Detect
  ↓
Prioritize
  ↓
Investigate
  ↓
Remediate
  ↓
Re-scan
  ↓
Verify PASS
```

The follow-up assessment demonstrated that correcting the underlying AWS configurations could resolve multiple related findings without remediating each check individually.

### Key Lesson

**A remediation is not complete until its effectiveness has been validated.**

Changing an AWS configuration is only the remediation step. A follow-up assessment provides evidence that the relevant security controls now pass.

---

## Troubleshooting

During the setup of Prowler Local Server, several issues were encountered involving Docker configuration, AWS credentials, and IAM Role assumption. The following notes document the root causes and final resolutions.

### 1. Docker Compose Configuration Error

**Problem**

Docker Compose returned a YAML configuration error when validating the custom override file.

**Root Cause**

The `docker-compose.override.yml` file contained incorrectly structured YAML, including indentation and mapping issues.

**Resolution**

Recreated the override file with the correct YAML structure and mounted the AWS configuration directory into the required Prowler services.

The final configuration used:

```yaml
services:
  api:
    volumes:
      - "${USERPROFILE}/.aws:/home/prowler/.aws:ro"

  worker:
    volumes:
      - "${USERPROFILE}/.aws:/home/prowler/.aws:ro"

  worker-beat:
    volumes:
      - "${USERPROFILE}/.aws:/home/prowler/.aws:ro"
```

**Validation**

```powershell
docker compose config
```

The command completed successfully, confirming that Docker Compose could validate the merged configuration.

### 2. AWS Credentials Not Available in Docker

**Problem**

Prowler Local Server returned:

```text
AWSAssumeRoleError[1012]
Unable to locate credentials
```

**Root Cause**

The Prowler containers could not access the AWS CLI configuration and source credentials stored on the Windows host.

**Resolution**

Mounted the host's `.aws` directory into the `api`, `worker`, and `worker-beat` containers as a read-only volume.

Configured the Local Server to use the source AWS CLI profile:

```text
AWS_PROFILE=default
```

**Validation**

Verified that the required configuration and credentials files were accessible inside the containers without displaying their secret contents.

The three services were checked individually:

- `api`
- `worker`
- `worker-beat`

### 3. AccessDenied During AssumeRole

**Problem**

AWS CLI returned an authorization error when attempting to assume the `ProwlerScan` role:

```text
AccessDenied
is not authorized to perform sts:AssumeRole
```

**Root Cause**

The IAM Role trust relationship did not allow the source IAM principal used by the AWS CLI to assume the role.

**Resolution**

Updated the CloudFormation stack's `IAMPrincipal` parameter to trust the intended source IAM principal rather than relying on the original default pattern.

The existing `ProwlerScan` role was updated instead of creating another role.

**Validation**

Executed:

```powershell
aws sts get-caller-identity --profile prowler-local
```

The command successfully returned an assumed-role ARN for `ProwlerScan`, confirming that the role assumption worked from the Windows host.

### 4. Repeated AssumeRole Attempt

**Problem**

Prowler Local Server returned an authorization error in which the caller was already an assumed `ProwlerScan` role but attempted to assume `ProwlerScan` again.

**Root Cause**

The Local Server was configured to use `prowler-local` as its `AWS_PROFILE`. That profile already performed role assumption, causing the application to use assumed-role credentials as its source identity.

**Resolution**

Changed the Local Server's environment configuration to:

```text
AWS_PROFILE=default
```

The `default` profile supplied the source credentials, while Prowler Local Server performed the role assumption using the IAM Role ARN and External ID configured in the AWS provider.

The separate `prowler-local` profile was retained for validating role assumption through the AWS CLI.

**Validation**

Restarted the Docker services and retried the provider connection. Prowler Local Server successfully connected to the AWS account.

### 5. Docker Engine Unavailable After Windows Shutdown

**Problem**

The Docker CLI could not connect to the Docker Desktop Linux engine after the laptop was shut down and restarted.

**Root Cause**

Docker Desktop and its Linux container engine were not running when the command was executed.

**Resolution**

Started Docker Desktop, waited for the engine to become available, and restarted the Prowler services from the project directory:

```powershell
docker compose up -d
```

**Validation**

```powershell
docker compose ps
```

Confirmed that the main Prowler services returned to their expected running or healthy states.

### Key Troubleshooting Lessons

- Docker containers must have access to the credential source required by their AWS authentication workflow.
- IAM Role assumption depends on both a valid source identity and an appropriate trust relationship.
- The source credential profile and the target assumed role serve different purposes.
- Docker Compose override files must be validated before restarting services.
- A Docker engine connection error does not necessarily indicate that the application or its persistent data is damaged.

## Project Conclusion

This project was created to gain hands-on experience with the complete AWS CSPM lifecycle using Prowler Local Server.

By introducing two controlled security misconfigurations, I practiced identifying findings, prioritizing risks, investigating root causes, and applying appropriate remediation. A follow-up scan confirmed that all four targeted checks passed after fixing the underlying issues.

**Key takeaway:** Effective cloud security is not just about detecting misconfigurations. It requires understanding the actual risk, addressing root causes, and validating that remediation works.