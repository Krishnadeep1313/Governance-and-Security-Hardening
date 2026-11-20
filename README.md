# ☁️ Lab 05: Implementing Governance and Security Hardening
 
![Azure](https://img.shields.io/badge/azure-%230072C6.svg?style=for-the-badge&logo=microsoftazure&logoColor=white)
![RBAC](https://img.shields.io/badge/RBAC-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Azure Policy](https://img.shields.io/badge/Azure_Policy-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Cost Management](https://img.shields.io/badge/Cost_Management-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![NIST CSF](https://img.shields.io/badge/NIST_CSF-5C2D91?style=for-the-badge)
 
**Author:** Krishnadeep Chudasama · **Estimated Time:** 60–75 minutes · **Difficulty:** Beginner
 
🎥 **WATCH ME COMPLETE THIS LAB HERE:**
 
<!-- TODO: replace this line with the Loom link once the video is recorded -->
 
## 📋 Table of Contents
- [Objective](#-objective)
- [Architecture Diagram](#-architecture-diagram)
- [Prerequisites](#-prerequisites)
- [Lab Variables](#-lab-variables)
- [Step-by-Step Instructions](#-step-by-step-instructions)
  - [Phase 1: Create the Resource Group](#phase-1-create-the-resource-group)
  - [Phase 2: RBAC, Create a User and Restrict Their Access](#phase-2-rbac-create-a-user-and-restrict-their-access)
  - [Phase 3: Verify Access (The Permission Denied Test)](#phase-3-verify-access-the-permission-denied-test)
  - [Phase 4: Azure Policy, Prevent Expensive Configurations](#phase-4-azure-policy-prevent-expensive-configurations)
  - [Phase 5: Test the Policy](#phase-5-test-the-policy)
  - [Phase 6: Cost Management, Set Up a Budget and Alerts](#phase-6-cost-management-set-up-a-budget-and-alerts)
- [NIST CSF Mapping](#-nist-csf-mapping)
- [Security Notes](#-security-notes)
- [Troubleshooting](#-troubleshooting)
- [Clean Up Resources](#-clean-up-resources)
---
 
## 🎯 Objective
In this lab, you will put three governance controls on a single resource group: **RBAC** (who can act), **Azure Policy** (what can be deployed) and a **Cost Management budget** (who finds out when spend drifts). You will create a "Junior Developer" with view-only access, block an expensive VM size even for the Owner account, and set a budget that emails you at 80% actual and 100% forecasted spend. Then you will try to break the first two controls on purpose to prove they work.
 
## 🏗️ Architecture Diagram
 
![Lab 05 architecture: Junior Developer blocked by the Reader role, Owner blocked by the Restrict-VM-Sizes policy, budget emails at 80% actual and 100% forecasted](images/lab05-architecture.png)
 
**How to read it:** Each row is a different control on `rg-lab05-gov-krish`. The Junior Developer tries to create a VM and RBAC blocks it (`AuthorizationFailed`). The Owner tries to deploy a VM size that is not on the allowed list and Policy blocks it (`RequestDisallowedByPolicy`). The budget watches spend and emails when a threshold is crossed. It warns, it never blocks.
 
## ✅ Prerequisites
- [x] Active Azure Subscription with Owner rights
- [x] A private/incognito browser window (to sign in as the Junior Developer)
- [x] Patience: a new policy assignment can take 10 to 30 minutes to take effect
## ⚙️ Lab Variables (Naming Convention)
Use these names to keep the deployment consistent:
* **Resource Group:** `rg-lab05-gov-krish` (East US)
* **Test User:** `junior-dev-krish@<your-tenant>.onmicrosoft.com`
* **Test User Display Name:** `Junior Developer`
* **Policy Assignment Name:** `Restrict-VM-Sizes`
* **Allowed VM sizes:** `Standard_B1s`, `Standard_B1ms`
* **Budget Name:** `Monthly-Lab-Budget` (CA$50 in my subscription, because it bills in CAD)
---
 
## 🚀 Step-by-Step Instructions
 
### Phase 1: Create the Resource Group
1. In the Azure portal, search for **Resource Groups** and click **Create**.
2. **Resource group name:** `rg-lab05-gov-krish`
3. **Region:** East US
4. Click **Review + create** -> **Create**.
Every control in this lab is scoped to this resource group, so nothing else in the subscription is affected.
 
### Phase 2: RBAC, Create a User and Restrict Their Access
 
**Step 1: Create the user in Microsoft Entra ID** *(NIST: Identify)*
1. Search for **Microsoft Entra ID** -> **Users** -> **+ New user** -> **Create new user**.
2. **User principal name:** `junior-dev-krish`
3. **Display name:** `Junior Developer`
4. Uncheck auto-generate password and set one you will remember.
5. Click **Review + create** -> **Create**.
![Entra ID Users list showing the Junior Developer account next to the admin account](images/lab05-01-entra-user.png)
 
A new user has **no access at all** until you grant it. Azure is deny-by-default.
 
**Step 2: Assign the Reader role at the resource group**
1. Open `rg-lab05-gov-krish` -> **Access control (IAM)**.
2. Click **+ Add** -> **Add role assignment**.
3. **Role:** `Reader`. **Members:** `Junior Developer`.
4. Click **Review + assign** twice.
![IAM on rg-lab05-gov-krish: Junior Developer holds Reader on This resource, Owner entries are inherited from the subscription](images/lab05-02-reader-role.png)
 
The scope column reads **This resource**, meaning the Reader role applies to this resource group only. The two Owner entries are inherited from the subscription.
 
### Phase 3: Verify Access (The Permission Denied Test)
1. Open a new **Incognito/Private** window and sign in at `portal.azure.com` as `junior-dev-krish@<your-tenant>.onmicrosoft.com`.
2. Open `rg-lab05-gov-krish` and click **+ Create**.
3. Start creating a resource (the SOP uses a Storage Account; I used a Virtual Machine) and click **Review + create**.
**Expected result:** a red validation error, `AuthorizationFailed`.
 
![Errors blade signed in as the Junior Developer: AuthorizationFailed on Microsoft.Resources/deployments/validate/action](images/lab05-03-authorization-failed.png)
 
The Reader role can view but not create. This is the **Protect** function working.
 
### Phase 4: Azure Policy, Prevent Expensive Configurations
1. Search for **Policy** -> **Assignments** -> **Assign policy**.
2. **Scope:** your subscription and `rg-lab05-gov-krish`.
3. **Policy definition:** search `Allowed virtual machine size SKUs` and select it.
4. **Assignment name:** `Restrict-VM-Sizes`
5. On the **Parameters** tab, uncheck "Only show parameters that need input or review" and select `Standard_B1s` and `Standard_B1ms`.
6. Click **Review + create** -> **Create**.
| Component | What it is |
|---|---|
| Policy definition | The rule (built in, by Microsoft): only the listed VM sizes are allowed |
| Policy assignment | Activates the rule on a scope. A definition does nothing until assigned |
| Allowed SKUs parameter | The values that make the rule specific: B1s and B1ms |
 
> ⏱️ Policy can take 10 to 30 minutes to propagate. If the test below passes validation, wait and retry.
 
### Phase 5: Test the Policy
1. From your **admin** account, open `rg-lab05-gov-krish` -> **+ Create** -> **Virtual machine**.
2. Pick Ubuntu Server and, for **Size**, choose one that is **not** on the list. I used `Standard_D2alds_v6`.
3. Click **Review + create**.
**Expected result:** "Validation failed", with `RequestDisallowedByPolicy` naming `Restrict-VM-Sizes`.
 
![Validation failed as the admin: Resource was disallowed by policy, Restrict-VM-Sizes. The highlighted price is about 0.108 USD per hour](images/lab05-04-policy-blocked.png)
 
This was my **Owner** account being blocked. RBAC can be worked around with higher rights; a Deny policy cannot. The portal estimated about US$0.108 an hour for that size, roughly US$79 a month if left running.
 
Then retry with `Standard_B1s`: validation should pass. (You do not need to finish the deployment.)
 
### Phase 6: Cost Management, Set Up a Budget and Alerts *(NIST: Detect)*
1. Open `rg-lab05-gov-krish` -> **Budgets** -> **+ Add**.
2. **Name:** `Monthly-Lab-Budget` · **Reset period:** Billing month · **Expiration:** one year out · **Amount:** 50.
3. **Alert condition 1:** Type **Actual**, **80%** of budget.
4. **Alert condition 2:** Type **Forecasted**, **100%** of budget.
5. **Alert recipients:** your email. Click **Create**.
![Budget summary: CA$50, resets monthly, alerts at 80% actual (CA$40) and 100% forecasted (CA$50), action group None](images/lab05-05-budget.png)
 
| Alert type | When it fires |
|---|---|
| Actual | After the money has already been spent (e.g. 80% = CA$40 of CA$50) |
| Forecasted | When Azure projects you will exceed the budget by the end of the period |
 
**What an alert looks like.** The email below is **not** from this lab's budget (which is at CA$0.00). It came from a separate budget on my account named `Monthly` with a $0.25 threshold, and fired when spend reached $1.08. It shows the shape of Detect and Respond: an email arrives asking me to review, and nothing else happens until a human acts.
 
![Azure budget alert email: evaluated amount 1.08 dollars against a 0.25 dollar threshold](images/lab05-06-budget-alert-email.png)
 
---
 
## 🧭 NIST CSF Mapping
 
![NIST CSF mapping of the lab: Identify, Protect, Detect, Respond, Recover with evidence and status](images/lab05-nist.png)
 
| Function | In this lab | Status |
|---|---|---|
| Identify | Created the Junior Developer user in Entra ID | Done |
| Protect | Reader role (RBAC) and the VM-size Deny policy | Done, both verified |
| Detect | Budget with actual and forecasted alerts | Configured |
| Respond | Alert email, then investigate in Cost analysis. No action group attached | Partly |
| Recover | Delete the test user and resource group | Pending |
 
---
 
## 🔐 Security Notes
Things worth knowing before you treat this as production-ready:
 
- **A budget does not stop spending.** It emails you. It never stops, deletes or resizes anything. To automate a response, attach an **action group** (my budget shows `Action group: None`).
- **Policy applies to everyone, including the Owner.** That is deliberate: governance cannot be bypassed by raising your own permissions.
- **Test both directions.** I proved the policy blocks a disallowed size. Also confirm an allowed size (`Standard_B1s`) passes validation.
- **Reader still reads.** The Junior Developer can view everything in the resource group, including configuration. Use Reader for visibility, not for hiding information.
- **Start with Audit in production.** A Deny policy can break existing deployments. Use the Audit effect first to see the impact, then switch to Deny.
- **Scope matters.** I scoped everything to one resource group. Assign at subscription or management group level only after the pattern is proven.
---
 
## 🛠️ Troubleshooting
- **Issue:** "The Junior Developer can still create resources."
  - **Fix:** The role was probably assigned at the wrong scope. Check `rg-lab05-gov-krish` -> Access control (IAM) -> Role assignments, and confirm Reader is listed there with scope **This resource**.
- **Issue:** "The policy is not blocking the VM."
  - **Fix:** Propagation takes 10 to 30 minutes. Wait and retry, then confirm `Restrict-VM-Sizes` appears under Policy -> Assignments with your resource group as the scope.
- **Issue:** "I cannot find Budgets in the resource group menu."
  - **Fix:** Search for **Cost Management + Billing** in the main search bar and open Budgets from there.
- **Issue:** "Incognito login asks me to set up MFA."
  - **Fix:** This is expected on first sign-in. Complete the setup.
- **Issue:** "Policy definition not found."
  - **Fix:** Search `virtual machine size` without quotes and pick the definition that mentions allowed sizes or SKUs.
- **Issue:** "Budget creation fails with a permissions error."
  - **Fix:** Confirm your account has the Owner or Cost Management Contributor role on the subscription.
---
 
## 🧹 Clean Up Resources
*Important: Always clean up unused resources and identities.*
 
1. Navigate to **Resource Groups** -> `rg-lab05-gov-krish` -> **Delete resource group**. Type the name to confirm. This also removes the policy assignment and budget scoped to it.
2. Go to **Microsoft Entra ID** -> **Users** -> `junior-dev-krish` -> **Delete**. Unused identities add to your attack surface.
3. Verify both are gone from Resource Groups and the Users list.
 
