# Variables in HCP Terraform (Terraform Cloud): Deep-Dive
## 1) What “variables” mean in HCP Terraform
Variables in HCP Terraform (Terraform Cloud) let you:
- **Customize module inputs** (Terraform *input* variables).
- **Control Terraform CLI behavior** (via **environment variables** like `TF_LOG`, `TF_CLI_ARGS_*`).
- **Provide provider credentials** (e.g., AWS, Azure, GCP) centrally and securely (often with variable **sets**).
You can define variables **per workspace** or reuse them across multiple workspaces and **Stacks** using **variable sets**. Workspace variables apply to *all runs* of that workspace unless explicitly overridden on a per-run basis using CLI flags or run-scoped settings.

---

## 2) Types of variables HCP Terraform understands

### A. Environment variables
- Exported into the shell of the remote worker VM before each run.  
- Typical uses: **provider credentials**, toggles like `TF_LOG`, concurrency knobs, etc.  
- Special case: **`TFE_PARALLELISM`** sets `-parallelism=<N>` for `plan/apply` (1–256, default 10). **Agents** don’t support `TFE_PARALLELISM`; use `TF_CLI_ARGS_plan` or `TF_CLI_ARGS_apply` instead.  
- You can also use **dynamic credentials** via env vars at workspace or variable set level to avoid long‑lived keys. 

### B. Terraform input variables
- Declared in HCL via `variable` blocks and referenced as `var.<name>`.  
- Values can be provided from: workspace variables, variable sets, `TF_VAR_*` env vars, `*.tfvars` files, and `-var`/`-var-file` CLI flags (see precedence below).  
- Mark inputs **sensitive** to redact in UI/logs; use complex types by enabling **HCL** evaluation for the variable in the UI. 

> **Note on HCL vs string in UI**: When adding a **Terraform** variable in the HCP Terraform UI, you can toggle **HCL** to allow structured values (maps, lists, objects) rather than plain strings—useful for complex inputs like tag maps. 

---

## 3) Scopes & reuse: Variable **sets**
- A **variable set** is a reusable collection of variables (both environment & Terraform) that you can apply to: the **entire org**, specific **projects**, specific **workspaces**, and **Stacks**.  
- Great for centralizing **provider auth**, shared settings (regions, tags), and guardrail defaults.  
- Manage via UI, **API**, or **`tfe` provider** resources.
**API highlights** (Variable Sets API):
- Create sets at **organization** or **project** scope.  
- `global=true` auto-applies to all current & future workspaces in the org.  
- Sets can have **priority** so their variables override others (including CLI), per API docs.  
- HCP Terraform **prevents conflicts** between global sets with duplicate variable names/types. 

> **Permissions**: Viewing variables requires **Read variables** on the workspace. Editing workspace variables needs **Read and write variables**. Managing org/project variable sets requires appropriate org/project permissions (e.g., Owners, Manage all projects/workspaces, or specific project roles). citeturn1search2

---

## 4) How to set variables (practical)
- **Workspace UI** → *Workspace* → **Variables** tab (add Terraform variables and env vars; mark **Sensitive**; toggle **HCL** for structured values).  
- **Variable sets UI** → Org/Project → Variable sets (create once, apply many).  
- **CLI (run-specific overrides)** → `-var`, `-var-file`, and local `TF_VAR_…` env vars; these override workspace/variable-set values for that specific run. 
**Examples (CLI)**:
```bash
# one-off override for a run
terraform plan -var="instance_type=t3.large" -var-file="prod.tfvars"

# using environment variables as input
export TF_VAR_region=eu-west-2
terraform apply
```

---


## 5) Variable **precedence** 

Terraform can receive values for the **same input variable** from many places. The value that finally wins depends on **where** and **how** you run (local vs. HCP Terraform remote), and whether the value comes from **Terraform inputs** (e.g., `TF_VAR_name`, `-var`) or plain **environment variables** (e.g., `AWS_ACCESS_KEY_ID`).

### 5.1 Input variables (the ones you declare with `variable` blocks)
**General Terraform order (highest → lowest)** for a given run:
1. **CLI flags**: `-var` and `-var-file` (later flags override earlier ones).  
2. **Explicit variable files** you pass via multiple `-var-file` flags (processed in the order you provide).  
3. **Auto-loaded files**: any `*.auto.tfvars` / `*.auto.tfvars.json` in the working directory (lexical order).  
4. **`terraform.tfvars` / `terraform.tfvars.json`** if present.  
5. **Environment** `TF_VAR_…` variables in the local shell (e.g., `TF_VAR_region`).  
6. **Default** value in the `variable` block (if any).  
This is the standard Terraform rule that “**later sources override earlier**”; CLI always wins. 

**HCP Terraform twist (remote runs):**
- When you run with the **CLI-driven remote workflow**, your local CLI translates your **run-scoped** inputs (items 1–5 above) into the run request sent to HCP Terraform. Those values therefore **override** any workspace/variable‑set values with the same key **for that run only**.
- **Workspace variables** and **Variable sets** supply values server-side. If a key exists in both, the **workspace variable** overrides the variable‑set value (unless you use API **priority** on a set; see below). You can still override both with run-scoped CLI inputs. 
- **Variable Sets API `priority=true`** (org/project set) can be used to make a set’s variables **override more specific scopes, including command-line** values. Use sparingly—this is powerful and can surprise operators if undocumented.

> **Rule of thumb:**
> **Run-scoped CLI** (`-var`, `-var-file`, `TF_VAR_…`) → **Workspace** → **Variable set(s)** → **Defaults**.  If a **priority** variable set exists, it can leapfrog to the top. citeturn1search2turn1search3

**Edge cases & notes**
- Multiple `*.auto.tfvars` files are loaded in **lexical order**; later ones win. JSON vs HCL forms follow the same precedence rule.
- In Terraform ≥ 0.12, **map/object** variables do **not merge**; the **last source fully overrides** the earlier value.

### 5.2 Plain environment variables (not `TF_VAR_…`)
These are variables the Terraform process can read from the shell, such as **provider auth** (`AWS_ACCESS_KEY_ID`, `ARM_CLIENT_ID`, etc.) or Terraform behavior toggles (`TF_LOG`, `TF_CLI_ARGS_*`, `TFE_PARALLELISM`).

- In **HCP Terraform remote runs**, the worker’s shell is populated from **workspace env vars** and any applied **variable sets**. If the same env var name exists in both, the **workspace env var** wins (most specific).
- **Variable Sets API `priority=true`** can force set values to override more specific scopes (again, use carefully).
- Local shell env like `TF_LOG` may still affect the **local** CLI behavior (e.g., logging) even in remote runs, but provider auth for remote execution **must** exist on the server side (workspace/variable-sets) because the provider initializes on the remote worker.

### 5.3 Visual cheat sheet
```
INPUT VARIABLES (same key)
Highest  ─┬─ Priority Variable Set (if enabled)             [HCP feature]
         ├─ Run-scoped CLI (-var, -var-file, TF_VAR_*)     [one run]
         ├─ Workspace variables                            [per workspace]
         ├─ Variable set(s)                                [org/project/workspace]
Lowest   ─┴─ Defaults in variable block

ENVIRONMENT VARS (same name)
Highest  ─┬─ Priority Variable Set (if enabled)
         ├─ Workspace env vars
Lowest   ─┴─ Variable set env vars
```

### 5.4 Practical recommendations
- Prefer **variable sets** for org-wide standards; allow **workspace overrides** only where justified. Document any **priority** sets prominently.
- Use **CLI overrides** for experiments and ephemeral changes, not for permanent policy. 
- Keep provider credentials in **workspace/variable-sets** (remote), not only local env, to avoid failed auth on workers. 

---


## 6) Parallelism tuning in HCP Terraform runs
- Use **`TFE_PARALLELISM`** (1–256, default 10) to adjust worker concurrency during remote runs.  
- Not supported by **agents**; use `TF_CLI_ARGS_plan`/`TF_CLI_ARGS_apply` to pass `-parallelism`.  
- Read the Terraform **Parallelism** docs before increasing it, and consider provider rate limits. 
---

## 7) Dynamic credentials (better than static keys)
- Configure per-run **temporary credentials** for supported providers using environment variables in the workspace or variable sets, eliminating manual key rotation.  
- Also supported when deploying **Stacks**. 
---

## 8) Managing variables at scale
- **UI** for ad-hoc edits.  
- **APIs**: Workspace Variables API (preferred; Variables API is deprecated), **Variable Sets API** for org/project/global reuse and priority.  
- **`tfe` provider** (e.g., `tfe_variable`, varset resources) for IaC management of your HCP Terraform org.

---

## 9) Security best practices & sensitive data
- Mark secrets as **Sensitive**; they are write‑only in UI and redacted in logs/outputs.  
- Prefer **dynamic credentials** or external secret stores over long‑lived keys.  
- Remember: secrets often still appear in **state**; keep state **encrypted** and access‑controlled.  
- Avoid committing secret `*.tfvars` to VCS. 

---

## 10) Hands‑on learning & references
- **Manage variables and variable sets** (official how‑to): UI, permissions, run‑specific overrides.
- **Use input variables** (language guide): authoring `variable` blocks, validation.
- **Variable Sets tutorial & sample repo**: manage multiple sets; precedence demos. 
---

## 11) Tips & Tricks (battle‑tested)
1. **Model shared creds with variable sets**: Create an org‑wide set for cloud provider auth; use **project‑level sets** for platform boundaries. Use **priority** on sets when you *must* override other sources consistently. citeturn1search3
2. **Prefer HCL input** for complex variables: enable **HCL** on Terraform variables in UI to pass objects/maps without JSON encoding gymnastics.
3. **Run‑specific experimentation**: use CLI `-var` flags in a *single run* to test changes without mutating workspace variables.
4. **Concurrency gotchas**: if providers throttle, dial back with `TFE_PARALLELISM`; for agents, inject `-parallelism` via `TF_CLI_ARGS_plan/apply`. 
5. **Sensitive handling**: mark secrets sensitive; prefer **dynamic credentials**; restrict variable visibility with least‑privilege permissions. 
6. **Version control safety**: keep non‑secret defaults in `*.auto.tfvars`; keep secrets out of VCS; consider Vault or cloud secret managers.
7. **Automate with the API**: bootstrap new projects by programmatically creating variable sets and assignments using the **Variable Sets API**. 

---

## 12) Quick MCQs (with answers)

**Q1.** Which of the following is **true** about `TFE_PARALLELISM`?  
A) Works with agents and worker VMs alike  
B) Only applies to remote worker VMs; agents must use `TF_CLI_ARGS_*` to pass `-parallelism`  
C) Default is 100  
D) Valid range is 0–1024  
**Answer:** **B**. It affects worker VMs; agents don’t support it. Default 10; valid range 1–256.

**Q2.** You need to reuse the same AWS credentials across 50 workspaces and allow a few workspaces to override the region. What should you use?  
A) Put variables in each workspace  
B) A **variable set** for credentials and region; override region in workspace as needed  
C) Only CLI `-var` flags  
**Answer:** **B**. Variable sets centralize shared values; workspace can override.

**Q3.** For a single run you want to try `instance_type=t3.large` without changing workspace variables. What’s best?  
A) Edit workspace variable and change it back later  
B) Add to org‑wide variable set  
C) `terraform plan -var="instance_type=t3.large"`  
**Answer:** **C**. Run‑scoped override via CLI. 

**Q4.** You must store a structured map of tags in a Terraform variable via the UI. What toggle makes this easy?  
A) Sensitive  
B) HCL  
C) Global  
**Answer:** **B**. Enable **HCL** to input objects/maps. 

**Q5.** Which statement about sensitive variables is **correct**?  
A) Marking a variable sensitive prevents it from being stored in state  
B) Sensitive values are redacted in outputs/logs, but may still exist in state; secure state access  
**Answer:** **B**. Redaction ≠ absence from state; protect state.
---

## 13) Handy snippets

### Example: Tags map via HCL in the UI
```hcl
# In the HCP Terraform UI, add a Terraform variable:
# key = common_tags, HCL = true
{
  environment = "prod",
  owner       = "platform-team",
  cost_center = "cc-1234"
}
```

### Example: Using TF_CLI_ARGS_* to pass flags when running via **agents**
```bash
# Agents ignore TFE_PARALLELISM; use TF_CLI_ARGS_* instead
export TF_CLI_ARGS_plan="-parallelism=5"
export TF_CLI_ARGS_apply="-parallelism=5"
```

---

## 14) Additional reading
- **Variables overview (HCP Terraform)** – the source for env vars, TFE_PARALLELISM, dynamic creds. 
- **Manage variables & variable sets** – UI workflow, permissions, run‑specific overrides. 
- **Variable Sets API** – create/org/project/global, priority, conflict rules.
- **Language guide: Input variables** – authoring `variable` blocks and validation. 
- **Tutorial: Manage multiple variable sets** – step‑by‑step walkthrough.



# Managing Variables in HCP Terraform   ( Different way to think )

HCP Terraform provides a centralized way to manage important values (variables) that can be reused across multiple projects and workspaces. By storing these values in one place, you can easily update them, and the changes automatically apply to all workspaces that use them. At the same time, you can override values for specific workspaces without affecting others.

---

## **Options for Managing Variables**

### **1. Workspace Variables**
Variables defined within a single workspace.  
- Scope: **Only that workspace**  
- Use case: Workspace-specific configurations.

### **2. Variable Sets**
Collections of variables that can be applied to multiple workspaces—or even globally across all workspaces in an organization.  
Variable sets are constrained to a single organization. You can't create variable sets that can be used across multiple HCP Terraform organizations.
- Scope:  
  - **Organization-wide**  
  - **Project-specific**  
  - **Selected workspaces**  
- Benefits:  
  - Enforce consistent configurations (e.g., cloud provider credentials, common tags).  
  - Apply globally to all current and future workspaces in the organization.

---

## **Run-Specific Variables**
For one-off changes or testing, you can use **run-specific variables** by passing values directly to Terraform commands:  
```bash
terraform plan -var="instance_type=t3.large" -var-file

