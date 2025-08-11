# GreyNoise SOAR Playbooks

## Introduction

This repository contains example Splunk SOAR playbooks that integrate with GreyNoise threat intelligence to help automate and enhance security operations.

These playbooks allow security teams to:  
- Automatically identify and contain IPs associated with known CVEs.
- Enrich artifacts with reputation data to reduce noise and prioritize threats.
- Automatically block or unblock IP addresses based on the GreyNoise IP Feed.

**Requirements:** Splunk SOAR version `6.4.1.361` or above.

**Note:** These playbooks are customizable and can be adapted to run on newly created artifacts or other trigger conditions based on the workflow.

---

## Overview of Playbooks

### Playbook 1: Network Containment

**Purpose:**  
This playbook retrieves IP addresses from GreyNoise associated with a specific CVE. To narrow down the query, it allows filtering based on GreyNoise IP classification and last seen date, and also allows the user to limit the total result count to fit in the limitations of some firewalls. The identified IPs are then automatically blocked using the configured firewall or security device (such as Fortigate or Zscaler).  The blocking action will need to be updated depending on the required integration.

**Inputs:**

- **CVE ID (required):** A valid Common Vulnerabilities and Exposures identifier (e.g., CVE-2023-12345). Used to fetch IPs from GreyNoise.

- **IP Classifications (optional):** Filters the returned IP list to only include IPs that match the specified classifications. When no option is included, IPs of any classification can exist in the returned list.  Options include:  
  - Benign (e.g., search engines)
  - Malicious (confirmed malicious)
  - Suspicious (probes, scanners)
  - Unknown (internet scanners that do not meet any other classification)
  Multiple values can be entered as a comma-separated list (e.g., Malicious,Suspicious). Leave blank to include all classifications.

- **Maximum IPs to Retrieve (optional):** This setting limits the number of IPs retrieved from the query that will be submitted for blocking. The default limit is `100`, but it should be adjusted according to the maximum number of IPs intended to block for the specified firewall.  

- **Time Filter (optional):** Limit IPs based on GreyNoise's last observation using the last_seen parameter. (valid options include but not limited to: 1d, 3d, 1w, 1m, today).  

**Workflow:**

- **GreyNoise GNQL Search**  
  Executes a GNQL query using the CVE and other filters to retrieve a list of matching IPs.
- **IP Blocking Logic**
  - If the asset only supports blocking one IP per action, the playbook uses the custom function `ip_by_ip` to loop over each IP and block it individually.
  - If the asset supports blocking multiple IPs, it use the `Generate Comma Separated IPs` block to format the IP list into a comma-separated string and sends it in a single action.

- **Blocking Action Examples**
  - **Fortigate (or similar):** Uses one-by-one IP blocking via `ip_by_ip`.
  - **Zscaler (or similar):** Uses a comma-separated string of IPs for batch blocking.

**Requirements:**
- Import the `ip_by_ip` custom function if using single-IP blocking assets.

**Key Notes:**
- Depending on the asset and action configured in the environment, remove the irrelevant portion of the playbook. For example, if using Zscaler, eliminate the Fortigate-specific flow, including the `Generate Comma Separated IPs` Block and any related steps.

---

### Playbook 2: Noise Elimination

**Purpose:**  
This playbook updates the severity level of an artifact based on GreyNoise reputation data for a single IP address.

**Inputs:**
This playbook utilizes an artifact field's value to execute the GreyNoise IP reputation action and obtain data on that IP. Ensure that the artifact IP path (e.g., `artifact:*.cef.sourceAddress`) is correctly set in the IP reputation action block of the playbook.

**To verify:**
1. Open any container and go to the Artifacts tab.
2. Click on an artifact and review the Details section to identify the relevant field (such as `sourceAddress` or `destinationAddress`).
3. Once identified the correct field, open the playbook and locate the IP reputation action block for GreyNoise.
4. Click on the IP parameter in that block. A pop-up will appear showing corresponding keys to the artifact.
5. Select the appropriate key. This will automatically populate the path in the format `artifact:*.cef.<field_name>`, such as `artifact:*.cef.sourceAddress`.

Mention a custom data path in the same format as above if the key is not present in the artifact pop-up.

**Workflow:**
- Extracts IPs from artifact CEF fields (e.g., `sourceAddress`, `destinationAddress`).
- Queries GreyNoise for each IP’s classification:
  - Malicious, Benign, Suspicious, or Unknown
- Determines if the IP has been Seen or Not Seen by GreyNoise sensors.
- Updates each artifact with two new fields:
  - `greynoiseClassification`
  - `greynoiseObservation`
- Adjusts artifact severity based on the classification:
  - Malicious → High
  - Suspicious → Medium
  - Benign → Low
  - Unknown or Not Seen → No severity change

**Key Notes:**
- In case the IP is Not Seen on GreyNoise, the playbook will skip adding the `greynoiseClassification` key to that artifact.

---

### Playbook 3: Network Containment Based on GreyNoise IP Feed

**Purpose:**  
This playbook blocks or unblocks IPs on Firewall based on the GreyNoise IP feed ingested via Webhook.

**Inputs:**  
This playbook requires no manual inputs or triggers. It runs automatically based on artifacts ingested from the GreyNoise IP feed Webhook. Follow the instructions in [Set Up Automatic Playbook Execution](#3-set-up-automatic-playbook-execution) to set up the playbook to run automatically. 

**Workflow:**

- Filter artifacts with tag `greynoise-ip-feed`.
- Based on the **new classification** of the IP, the playbook either blocks or unblocks the IP:
  - If classification is `Malicious`, block the IP
  - If classification is `Benign`, unblock the IP

This logic can be customized. For example:

- If webhook only ingests `Malicious` IPs, retain only the block logic.
- To block `Suspicious` IPs, modify the `Check New IP Classification` decision block to include `Suspicious` classification.

**Key Note:**  
This example uses **FortiGate** for blocking/unblocking. Modify the playbook as needed based on the environment and firewall capabilities.

---

## Post-Import Setup in Splunk SOAR

After importing the playbooks into the Splunk SOAR environment, complete the following configuration steps:

### 1. Configure Required Assets

**GreyNoise Asset**
- Go to Apps > GreyNoise > Configure New Asset
- Set:
  - `api_key`: GreyNoise enterprise API key

**Firewall/Security Asset (e.g., Zscaler, Fortigate)**
- Determine whether the asset supports blocking one IP or multiple IPs.
- Configure credentials (username, password, API token, base URL)
- Verify that the asset’s API is accessible from the SOAR instance.

---

### 2. Import Custom Function (if needed)

If the blocking asset supports one IP per action, use the `ip_by_ip` custom function:

- From the **Home** menu, go to **Custom Functions**
- Check if `ip_by_ip` exists
- If not, import it from a `.tgz` file before running the playbook

---

### 3. Set Up Automatic Playbook Execution

To run playbook automatically on newly ingested artifacts:

- From the **Home** menu, go to **Playbooks**.
- Click the name of the playbook to configure.
- Open the playbook settings by clicking the **⚙️ Settings** button in the top-right corner.
  - Set **Operates on** to the correct label that matches the incoming artifacts.
  - Set **Run automatically when** to `Artifacts created`.
  - Enable the **Active** toggle.
  - *(Optional)* Enable the **Logging** toggle to capture debug logs.
- Click **Save** to apply the settings.

---

### 4. Set Appropriate Permissions

Ensure that the user or role running the playbook has:
- Access to required assets and actions
- Sufficient role permissions to update artifact fields or mark severity
