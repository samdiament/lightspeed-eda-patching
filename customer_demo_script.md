# Red Hat Lightspeed & Satellite Closed-Loop CVE Remediation Demo Script

This runbook provides a step-by-step guide for conducting a live demonstration of automated, event-driven CVE remediation using **Red Hat Lightspeed**, **Ansible Automation Platform (AAP) 2.7**, **Red Hat Satellite 6.19**, and **RHEL**.

---

## 1. Architecture Overview

```
[ RHEL Target Host ] --(insights-client)--> [ Red Hat Lightspeed ]
        ^                                            |
        |                                     (System Vulnerability Webhook)
   (Patch / Sync)                                    v
        |                                   [ AAP 2.7 EDA Rulebook ]
        |                                            |
   [ Red Hat Satellite ] <--(Content Prep)-- [ 3-Stage Workflow Template ]
   (Content View & CVs)                             |
                                           [ Stage 2: Human Approval ]
```

* **RHUI Target Nodes:** `rhel9-node01.samboxaws.org`, `rhel10-node01.samboxaws.org`
* **Satellite Target Nodes:** `rhel9-sat01.samboxaws.org`, `rhel10-sat01.samboxaws.org`
* **Infrastructure Host:** `satellite.samboxaws.org`
* **AAP 2.7 Host:** `aap.samboxaws.org`

---

## 2. Pre-Call Setup Checklist (5–10 Minutes Before Presentation)

- [ ] **Power on Demo Infrastructure:**
  If the environment was shut down, spin up the VMs using the management playbook:
  ```bash
  ansible-playbook manage_demo_vms.yml -e vm_state=running
  ```
- [ ] **Sync Dynamic Inventories in AAP:**
  In **Automation Execution → Infrastructure → Inventories**, trigger a manual sync on both your **RHUI AWS** and **Red Hat Satellite** inventory sources to update `ansible_host` IP bindings.
- [ ] **Verify EDA Rulebook Activation:**
  In **Automation Decisions → Rulebook Activations**, confirm the activation running `rulebooks/lightspeed_patch.yml` is **Running** and healthy.
- [ ] **Verify Clean Baseline on Target Node:**
  Confirm the node selected for the demo (e.g., `rhel9-sat01.samboxaws.org`) shows **0 Important/Critical CVEs** in **Red Hat Lightspeed → Inventory**.
- [ ] **Open Browser Tabs in Presentation Order:**
  * **Tab 1:** Red Hat Lightspeed → Inventory (`console.redhat.com/ansible/lightspeed`)
  * **Tab 2:** Red Hat Lightspeed → Settings → Notifications → Event Log
  * **Tab 3:** AAP 2.7 → Automation Decisions → Rule Audit
  * **Tab 4:** AAP 2.7 → Automation Execution → Jobs (or Workflow Visualizer)
  * **Tab 5:** Red Hat Satellite → Content → Content Views (if highlighting Satellite content promotion)

---

## 3. Live Demo Script

### Act 1: Healthy Baseline & Triggering the Vulnerability (~2 min)

**Narrative:**
> *"Right now, our RHEL target host is fully patched and reporting zero active vulnerabilities to Red Hat Lightspeed. To simulate a real-world scenario where a zero-day or new CVE is identified in production, I am going to introduce a vulnerable package version on the host and run a system check-in."*

**Target Host Terminal (`rhel9-sat01.samboxaws.org`):**
```bash
# Check available package versions in Satellite Content View
sudo dnf --showduplicates list libxml2

# Downgrade to trigger vulnerable package state
sudo dnf downgrade -y libxml2

# Verify unpatched CVEs exist locally
sudo dnf updateinfo list cves --sec-severity=Important --sec-severity=Critical

# Upload updated system profile to Red Hat Lightspeed
sudo insights-client
```

**Screen Action:**
1. Switch to **Tab 1 (Red Hat Lightspeed Inventory)**. Refresh the host view for `rhel9-sat01.samboxaws.org` to show the newly detected CVEs.
2. Switch to **Tab 2 (Notifications Event Log)** to show the `System vulnerability` event generated and dispatched to AAP.

---

### Act 2: Event-Driven Automation & Satellite Content Preparation (~2 min)

**Narrative:**
> *"Rather than running an uncoordinated script, Lightspeed emits a webhook payload into Event-Driven Ansible. AAP evaluates the incoming CVE metadata and launches a multi-stage remediation workflow. Stage 1 automatically queries Red Hat Satellite to ensure that the required errata and package content are available in the host's active Content View and Lifecycle Environment before any patching is attempted."*

**Screen Action:**
1. Switch to **Tab 3 (AAP Rule Audit)**. Point out the incoming event matching the rule `Remediate Lightspeed Vulnerability Event`.
2. Switch to **Tab 4 (AAP Jobs / Workflow Visualizer)**. Show **Node 1 (`satellite_prepare_content.yml`)** executing:
   * Identifying missing/unpromoted CVE errata.
   * Triggering incremental Content View promotion if required.
   * Publishing a structured summary for the upcoming approval gate.

---

### Act 3: Human-in-the-Loop Governance (~1–2 min)

**Narrative:**
> *"Automated content prep ensures technical readiness, but critical production environments require operational oversight. The workflow pauses at Stage 2, giving the operations or security team full visibility into what will be changed before granting sign-off."*

**Screen Action:**
1. Focus on the **Approval Node** in the Workflow Visualizer.
2. Open the approval prompt to show the host, targeted CVEs, and Satellite triage status.
3. Click **Approve** to proceed to remediation.

---

### Act 4: Automated Patching & Closed-Loop Verification (~2 min)

**Narrative:**
> *"With approval granted, Stage 3 executes our targeted DNF remediation playbook against the host. Once the package update completes, the playbook triggers `insights-client` to close the loop—updating both Red Hat Satellite and Red Hat Lightspeed in real time."*

**Screen Action:**
1. Watch **Node 3 (`patch_rhel.yml`)** execute and complete with status **Successful**.
2. Switch back to **Tab 1 (Red Hat Lightspeed Inventory)** and refresh the host view. Point out that the host has returned to **0 CVEs**, confirming end-to-end remediation.

---

## 4. Post-Demo Reset & Cleanup

To reset the node back to a baseline state for subsequent runs or shut down the demo environment:

### Reset Host Baseline
```bash
# SSH into target node and ensure latest updates are applied
sudo dnf upgrade -y libxml2
sudo insights-client
```

### Power Off Infrastructure (End of Day / Weekend)
```bash
ansible-playbook manage_demo_vms.yml -e vm_state=stopped
```

---

## 5. Live Troubleshooting Cheat Sheet

| Symptom | Probable Cause | Quick Fix |
| :--- | :--- | :--- |
| **No CVEs appear in Lightspeed after `insights-client`** | Filters active or report upload incomplete. | Clear search/filter chips on the **Vulnerability** tab. Verify `insights-client` ended with `Successfully uploaded report`. |
| **Event fired in Lightspeed, but AAP didn't trigger** | Rulebook activation inactive or payload mismatch. | Check **Automation Decisions → Rulebook Activations**. Verify event payload includes matching hostname and `reported_cve`. |
| **Workflow fails on launch with `limit` or `extra_vars` error** | "Prompt on launch" not checked on Workflow/Job template. | Open the **Workflow Job Template** and **`Patch RHEL Host` Job Template** in AAP; verify **Prompt on launch** is enabled for both **Variables** and **Limit**. |
| **Satellite stage fails on dynamic inventory host** | `ansible_host` pointing to FQDN instead of private IP. | Re-sync the **Red Hat Satellite** inventory source in AAP and confirm `compose: ansible_host: foreman_ipv4` is present in source variables. |