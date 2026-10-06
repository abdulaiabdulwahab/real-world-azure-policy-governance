# real-world-azure-policy-governance

# Real-World Azure Policy and Governance

## Project Overview

This project demonstrates how **Azure Policy and Governance** can be used to create an organizational governance baseline for Azure resources.

The project uses custom policies, an Azure Policy initiative, enforcement modes, exemptions, managed identities, and remediation to simulate how governance controls are implemented in a real production environment.

---

## Project Objectives

The governance baseline in this project ensures that:

- Azure resources are deployed only in approved regions
- Resources contain required governance tags
- Multiple policies are grouped into an initiative
- Policies are tested before enforcement
- Non-compliant deployments can be blocked
- Policy exceptions can be managed using exemptions
- Existing resources can be corrected using remediation

---

## Architecture

```text
Developer / Azure CLI / Terraform
              |
              v
      Azure Resource Manager
              |
              v
         Azure Policy
              |
      +-------+-------+
      |               |
Allowed Region?   Required Tag?
      |               |
      +-------+-------+
              |
              v
        Policy Initiative
              |
       +------+------+
       |             |
   Compliant     Non-Compliant
       |             |
       v             v
    Deploy          Deny
```

---

## Technologies Used

- Microsoft Azure
- Azure Policy
- Azure Policy Initiatives
- Azure CLI
- Azure Resource Groups
- Azure Storage Accounts
- Managed Identities
- Azure RBAC
- Git
- GitHub

---

## Project Structure

```text
real-world-azure-policy-governance/
│
├── policies/
│   ├── allowed-locations/
│   │   ├── rule.json
│   │   └── parameters.json
│   │
│   └── required-tag/
│       ├── rule.json
│       └── parameters.json
│
└── initiatives/
    ├── definitions.json
    └── parameters.json
```

---

## Governance Requirements

The simulated production environment uses the following requirements:

```text
Approved Azure Region:
canadacentral

Required Tag:
CostCenter
```

Resources that do not meet these requirements can be denied by Azure Policy.

---

## Step 1 — Configure Project Variables

```bash
RG="rg-policy-governance-prod"
LOCATION="canadacentral"

LOCATION_POLICY="corp-allowed-locations"
TAG_POLICY="corp-required-tag"

INITIATIVE="corp-production-governance"
INIT_ASSIGNMENT="production-governance-baseline"

SUB_ID=$(az account show \
  --query id \
  --output tsv)

SCOPE="/subscriptions/$SUB_ID/resourceGroups/$RG"
```

---

## Step 2 — Create the Resource Group

```bash
az group create \
  --name "$RG" \
  --location "$LOCATION" \
  --tags \
    Environment=prod \
    CostCenter=CC1001 \
    Owner=PlatformTeam
```

---

## Step 3 — Create the Allowed Locations Policy

The policy ensures resources are deployed only in approved Azure regions.

Example policy rule:

```json
{
  "if": {
    "allOf": [
      {
        "field": "location",
        "notIn": "[parameters('allowedLocations')]"
      },
      {
        "field": "location",
        "notEquals": "global"
      }
    ]
  },
  "then": {
    "effect": "deny"
  }
}
```

Create the policy:

```bash
az policy definition create \
  --name "$LOCATION_POLICY" \
  --display-name "Corporate Allowed Locations" \
  --description "Restricts Azure resources to approved regions." \
  --mode Indexed \
  --rules ./policies/allowed-locations/rule.json \
  --params ./policies/allowed-locations/parameters.json \
  --version 1.0.0
```

---

## Step 4 — Create the Required Tag Policy

This policy requires resources to contain a governance tag.

Example:

```text
CostCenter
```

Create the definition:

```bash
az policy definition create \
  --name "$TAG_POLICY" \
  --display-name "Corporate Required Tag" \
  --description "Requires the specified governance tag." \
  --mode Indexed \
  --rules ./policies/required-tag/rule.json \
  --params ./policies/required-tag/parameters.json \
  --version 1.0.0
```

---

## Step 5 — Retrieve the Policy IDs

```bash
LOCATION_POLICY_ID=$(az policy definition show \
  --name "$LOCATION_POLICY" \
  --query id \
  --output tsv)
```

```bash
TAG_POLICY_ID=$(az policy definition show \
  --name "$TAG_POLICY" \
  --query id \
  --output tsv)
```

Verify:

```bash
echo "$LOCATION_POLICY_ID"
echo "$TAG_POLICY_ID"
```

---

## Step 6 — Create the Policy Initiative

The initiative groups both governance policies together:

```text
Corporate Production Governance Baseline
│
├── Allowed Locations
│
└── Required CostCenter Tag
```

Create the initiative:

```bash
az policy set-definition create \
  --name "$INITIATIVE" \
  --display-name "Corporate Production Governance Baseline" \
  --description "Baseline location and tagging controls for production resources." \
  --definitions @initiatives/definitions.json \
  --params @initiatives/parameters.json \
  --version 1.0.0
```

Verify:

```bash
az policy set-definition show \
  --name "$INITIATIVE" \
  --output jsonc
```

---

## Step 7 — Assign the Initiative Without Enforcement

The initiative is first deployed using `DoNotEnforce`.

```bash
az policy assignment create \
  --name "$INIT_ASSIGNMENT" \
  --display-name "Production Governance Baseline" \
  --policy-set-definition "$INITIATIVE" \
  --scope "$SCOPE" \
  --params '{
    "allowedLocations":{"value":["canadacentral"]},
    "requiredTagName":{"value":"CostCenter"}
  }' \
  --enforcement-mode DoNotEnforce
```

This allows compliance to be evaluated without blocking resources.

---

## Step 8 — Test a Non-Compliant Resource

Create a storage account in an unapproved region without the required tag:

```bash
BAD_STG="stgovbad$RANDOM$RANDOM"

az storage account create \
  --name "$BAD_STG" \
  --resource-group "$RG" \
  --location eastus \
  --sku Standard_LRS
```

Because enforcement is disabled, the deployment can succeed.

Trigger a policy scan:

```bash
az policy state trigger-scan \
  --resource-group "$RG"
```

Check compliance:

```bash
az policy state list \
  --resource-group "$RG" \
  --query "[].{Resource:resourceId,State:complianceState,Policy:policyDefinitionReferenceId}" \
  --output table
```

---

## Step 9 — Enable Enforcement

```bash
az policy assignment update \
  --name "$INIT_ASSIGNMENT" \
  --scope "$SCOPE" \
  --enforcement-mode Default
```

Azure Policy can now block non-compliant deployments.

---

## Step 10 — Test Region Enforcement

Attempt to deploy in `eastus`:

```bash
az storage account create \
  --name "stregion$RANDOM$RANDOM" \
  --resource-group "$RG" \
  --location eastus \
  --sku Standard_LRS \
  --tags CostCenter=CC1001
```

Expected result:

```text
RequestDisallowedByPolicy
```

---

## Step 11 — Test Tag Enforcement

Attempt to deploy without the `CostCenter` tag:

```bash
az storage account create \
  --name "sttag$RANDOM$RANDOM" \
  --resource-group "$RG" \
  --location canadacentral \
  --sku Standard_LRS
```

The deployment should be denied.

---

## Step 12 — Deploy a Compliant Resource

```bash
GOOD_STG="stgovgood$RANDOM$RANDOM"

az storage account create \
  --name "$GOOD_STG" \
  --resource-group "$RG" \
  --location canadacentral \
  --sku Standard_LRS \
  --tags \
    CostCenter=CC1001 \
    Application=Payments
```

This deployment should succeed because it meets both governance requirements.

---

## Step 13 — Policy Exemptions

Azure Policy exemptions can be used when a resource needs an approved temporary exception.

For example, a temporary exemption can allow a resource to bypass the location policy without disabling the entire governance initiative.

```text
Initiative
│
├── Allowed Locations → Exempted
│
└── Required Tag      → Still Enforced
```

This provides more controlled governance than removing or disabling policies.

---

## Step 14 — Remediation

A `Modify` policy was also used to demonstrate how Azure Policy can correct existing resources.

The built-in policy:

```text
Inherit a tag from the resource group if missing
```

was assigned using a system-assigned managed identity.

The managed identity was granted:

```text
Tag Contributor
```

A remediation task was then created:

```bash
az policy remediation create \
  --name remediate-environment-tags \
  --policy-assignment inherit-environment-from-rg \
  --resource-group "$RG"
```

Check remediation status:

```bash
az policy remediation show \
  --name remediate-environment-tags \
  --resource-group "$RG" \
  --output jsonc
```

This demonstrates how Azure Policy can correct existing non-compliant resources instead of only blocking new deployments.

---

## Governance Types Demonstrated

```text
Preventive Governance
        |
        └── Deny

Detective Governance
        |
        └── Compliance Evaluation

Corrective Governance
        |
        └── Modify + Remediation
```

---

## Troubleshooting

### Failed to Parse `--definitions`

If Azure CLI reports:

```text
Failed to parse '--definitions' argument
```

verify the file:

```bash
cat initiatives/definitions.json
```

The file should contain only valid JSON.

Validate it with:

```bash
python -m json.tool initiatives/definitions.json
```

---

### Policy Is Not Blocking Resources

Check the enforcement mode:

```bash
az policy assignment show \
  --name "$INIT_ASSIGNMENT" \
  --scope "$SCOPE" \
  --query enforcementMode
```

If it returns:

```text
DoNotEnforce
```

enable enforcement:

```bash
az policy assignment update \
  --name "$INIT_ASSIGNMENT" \
  --scope "$SCOPE" \
  --enforcement-mode Default
```

---

### No Policy Assignment Found During Remediation

Using the full Azure resource ID in Git Bash can sometimes cause path-conversion issues.

Instead of using the resource ID:

```bash
--policy-assignment "$INHERIT_ASSIGNMENT_ID"
```

use the assignment name:

```bash
--policy-assignment inherit-environment-from-rg
```

Example:

```bash
az policy remediation create \
  --name remediate-environment-tags \
  --policy-assignment inherit-environment-from-rg \
  --resource-group "$RG"
```

---

### Compliance Has Not Updated

Trigger another evaluation:

```bash
az policy state trigger-scan \
  --resource-group "$RG"
```

Then:

```bash
az policy state summarize \
  --resource-group "$RG"
```

---

## Git Version Control

Initialize the repository:

```bash
git init
```

Add the files:

```bash
git add .
```

Commit:

```bash
git commit -m "Add Azure Policy governance baseline"
```

Connect to GitHub:

```bash
git branch -M main

git remote add origin \
  https://github.com/<USERNAME>/real-world-azure-policy-governance.git

git push -u origin main
```

This demonstrates the concept of **Policy as Code**, where governance definitions are stored and version-controlled alongside other infrastructure code.

---

## Cleanup

Delete the remediation:

```bash
az policy remediation delete \
  --name remediate-environment-tags \
  --resource-group "$RG"
```

Delete the tag inheritance assignment:

```bash
az policy assignment delete \
  --name inherit-environment-from-rg \
  --scope "$SCOPE"
```

Delete the governance initiative assignment:

```bash
az policy assignment delete \
  --name "$INIT_ASSIGNMENT" \
  --scope "$SCOPE"
```

Delete the initiative:

```bash
az policy set-definition delete \
  --name "$INITIATIVE"
```

Delete the custom policies:

```bash
az policy definition delete \
  --name "$LOCATION_POLICY"
```

```bash
az policy definition delete \
  --name "$TAG_POLICY"
```

Delete the resource group:

```bash
az group delete \
  --name "$RG" \
  --yes \
  --no-wait
```

---

## What I Learned

This project provided hands-on experience with:

- Azure Policy definitions
- Azure Policy initiatives
- Policy assignments
- Policy parameters
- Policy enforcement
- `DoNotEnforce` mode
- Policy compliance
- Policy exemptions
- Managed identities
- Azure RBAC
- Policy remediation
- Resource tagging governance
- Regional deployment restrictions
- Policy as Code
- Azure CLI
- Git and GitHub

---

## Key Takeaway

Azure Policy provides centralized controls for enforcing organizational standards across Azure environments.

This project demonstrated a realistic governance lifecycle:

```text
Create Policies
      ↓
Store Policies as Code
      ↓
Create Initiative
      ↓
Assign with DoNotEnforce
      ↓
Evaluate Compliance
      ↓
Enable Enforcement
      ↓
Deny Non-Compliant Resources
      ↓
Create Approved Exceptions
      ↓
Remediate Existing Resources
```

This approach allows organizations to implement consistent, scalable, and automated Azure governance while still supporting controlled exceptions and remediation.