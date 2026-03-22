# 🛡️ Building a Privacy-First SOC Pipeline (Personal Privacy Guard)

## 📌 Project Overview
In modern Security Operations Centers (SOCs), balancing network visibility with user privacy is a critical compliance requirement (e.g., GDPR). This project demonstrates the implementation of a **Personal Privacy Guard (PPG)** pipeline. 

Instead of sending raw logs directly to the SIEM, logs are intercepted by a Logstash gateway. This gateway uses Regular Expressions (Regex) to detect and redact Personally Identifiable Information (PII)—such as email addresses and specific host IPs—before they are ingested and stored by Wazuh.

### 🎯 Key Objectives
* Differentiate between DLP (Data Loss Prevention) and PPG (Personal Privacy Guard).
* Build an ingestion gateway to sanitize logs in transit.
* Configure custom SIEM rules to alert on sanitized data.
* Troubleshoot log parsing and network ingestion issues between the Elastic stack and Wazuh.

---

## 🏗️ Lab Environment & Architecture

[Image of a network architecture diagram showing an Alpine Linux workstation sending logs to an Ubuntu Logstash gateway, which sanitizes and forwards them to a Wazuh SIEM]

The lab consists of three distinct virtual machines operating on a local subnet:
* **User Workstation (Source):** Alpine Linux (`192.168.137.143`)
* **Privacy Gateway (PPG):** Ubuntu with Logstash (`192.168.137.131`)
* **SIEM / Storage:** Wazuh Pre-installed OVA (`192.168.137.144`)

**Data Flow:** Alpine Source (TCP 5044) -> Ubuntu Logstash (Sanitization) -> Wazuh Manager (UDP 514)

---

## 🚀 Step-by-Step Implementation & Verification

### Phase 1: Setting Up the SIEM Destination (Wazuh)
Since Wazuh was deployed via a pre-installed OVA, the core installation was handled by the virtualization platform. The goal here is to prepare Wazuh to receive and understand custom remote logs.

**1. Enable Remote Syslog Ingestion**
By default, Wazuh does not listen for external syslog. I configured it to accept logs strictly from the Logstash gateway.
* Open the configuration: `sudo nano /var/ossec/etc/ossec.conf`
* Add the remote syslog block:
  ```xml
  <remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>192.168.137.131/32</allowed-ips>
  </remote>
