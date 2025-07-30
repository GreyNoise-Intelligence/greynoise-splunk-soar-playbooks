# GreyNoise SOAR Playbooks

## Introduction
This repository contains example Splunk SOAR playbooks that integrate with GreyNoise threat intelligence to help automate and enhance your security operations.  
These playbooks allow security teams to:  
Automatically identify and contain IPs associated with known CVEs.

Enrich artifacts with reputation data to reduce noise and prioritize threats effectively.

**Requirements:** Splunk SOAR version `6.4.1` or above 

**Note:** These playbooks are customizable and can be adapted to run on newly created artifacts or other trigger conditions based on your workflow.

---

## Overview of Playbooks

### Playbook 1: Network Containment

**Purpose:**  
This playbook retrieves IP addresses from GreyNoise that are associated with a specific CVE. It allows filtering based on classification, last seen time, and result limit to narrow down the query. The identified IPs are then automatically blocked using your configured firewall or security device (such as Fortigate or Zscaler).

**Inputs:**

- **CVE ID (required):** A valid Common Vulnerabilities and Exposures identifier (e.g., CVE-2023-12345). Used to fetch IPs from GreyNoise.

- **IP Classifications (optional):** Filter based on IP type. Options include:
  - Benign (e.g., search engines)
  - Malicious (confirmed malicious)
  - Suspicious (probes, scanners)
  - Unknown (unclassified IPs)  
  Multiple values can be entered as a comma-separated list (e.g., Malicious,Suspicious). Leave blank to include all classifications.

- **Maximum IPs to Retrieve (optional):** Limits the number of IPs retrieved and blocked. Default is 100.

- **Time Filter (optional):** Restrict IPs based on when GreyNoise last observed them (e.g., 1d, 1w, today, yesterday).

**Workflow:**

- **GreyNoise GNQL Search**  
  Executes a GNQL query using the CVE and other filters to retrieve a list of matching IPs with metadata.

- **IP Blocking Logic**
  - If the asset only supports blocking one IP per action, the playbook uses the custom function `ip_by_ip` to loop over each IP and block it individually.
  - If the asset supports blocking multiple IPs, it use the `Generate Comma Separated IPs` block to format the IP list into a comma-separated string and sends it in a single action.

- **Blocking Action Examples**
  - **Fortigate (or similar):** Uses one-by-one IP blocking via `ip_by_ip`.
  - **Zscaler (or similar):** Uses a comma-separated string of IPs for batch blocking.

**Requirements:**
- Import the `ip_by_ip` custom function if using single-IP blocking assets.

**Key Notes:**
- Depending on the asset and action configured in your environment, remove the irrelevant portion of the playbook. For example, if you're using Zscaler, you can eliminate the Fortigate-specific flow, including the `Generate Comma Separated IPs` Block and any related steps.

---

### Playbook 2: Noise Elimination

**Purpose:**  
This playbook classifies event severity based on GreyNoise reputation data for a single IP address.

**Inputs:**
This playbook uses the value of an artifact field to perform IP reputation. Ensure that the artifact IP path (e.g., `artifact:*.cef.sourceAddress`) is correctly set in the IP reputation action block of the playbook.

**To verify:**
1. Open any container and go to the Artifacts tab.
2. Click on an artifact and review the Details section to identify the relevant field (such as `sourceAddress` or `destinationAddress`).
3. Once you've identified the correct field, open the playbook and locate the IP reputation action block for GreyNoise.
4. Click on the IP parameter in that block. A pop-up will appear showing corresponding keys to the artifact.
5. Select the appropriate key. This will automatically populate the path in the format `artifact:*.cef.<field_name>`, such as `artifact:*.cef.sourceAddress`.

You can also mention a custom data path in the same format as above if the key is not present in the artifact pop-up.

**Workflow:**
- Extracts IPs from artifact CEF fields (e.g., `sourceAddress`, `destinationAddress`).
- Queries GreyNoise for each IP’s classification:
  - Malicious, Benign, Suspicious, or Unknown
- Determines if the IP has been Seen or Not Seen by GreyNoise sensors.
- Updates each artifact with two new fields:
  - `greynoise_classification`
  - `greynoise_observation`
- Adjusts artifact severity based on the classification:
  - Malicious → High
  - Suspicious → Medium
  - Benign → Low
  - Unknown or Not Seen → No severity change

**Key Notes:**
- In case the IP is Not Seen on GreyNoise, the playbook will skip adding the `greynoise_classification` key to that artifact.

---

## Post-Import Setup in Splunk SOAR

After importing the playbooks into your Splunk SOAR environment, complete the following configuration steps:

### 1. Configure Required Assets

**GreyNoise Asset**
- Go to Apps > GreyNoise > Configure New Asset
- Set:
  - `api_key`: Your GreyNoise enterprise API key
  - `base_url`: Optional, only if using a custom endpoint

**Firewall/Security Asset (e.g., Zscaler, Fortigate)**
- Determine whether the asset supports blocking one IP or multiple IPs.
- Configure credentials (username, password, API token, base URL)
- Verify that the asset’s API is accessible from your SOAR instance.

---

### 2. Import Custom Function (if needed)

If your blocking asset supports one IP per action, you must use the `ip_by_ip` custom function:

- Go to Playbooks > Custom Functions
- Check if `ip_by_ip` exists
- If not, import it from a `.tgz` file before running the playbook

---

### 3. Set Appropriate Permissions

Ensure that the user or role running the playbooks has:
- Access to required assets and actions
- Sufficient role permissions to update artifact fields or mark severity