🛡️ Building a Privacy-First SOC Pipeline (Personal Privacy Guard)
📌 Project Overview
In modern Security Operations Centers (SOCs), balancing network visibility with user privacy is a critical compliance requirement (e.g., GDPR). This project demonstrates the implementation of a Personal Privacy Guard (PPG) pipeline.

Instead of sending raw logs directly to the SIEM, logs are intercepted by a Logstash gateway. This gateway uses Regular Expressions (Regex) to detect and redact Personally Identifiable Information (PII) such as email addresses and specific host IPs before they are ingested and stored by Wazuh.

🎯 Key Objectives
Differentiate between DLP (Data Loss Prevention) and PPG (Personal Privacy Guard).

Build an ingestion gateway to sanitize logs in transit.

Configure custom SIEM rules to alert on sanitized data.

Troubleshoot log parsing and network ingestion issues between Elastic stack components and Wazuh.

🏗️ Architecture & Tech Stack
Log Generator (Source): Alpine Linux / Ubuntu (simulating application logs via nc).

Privacy Gateway (PPG): Logstash running on Ubuntu.

SIEM / Storage: Wazuh Manager & Dashboard.

Protocols: TCP (Port 5044 for log ingestion), UDP Syslog (Port 514 for SIEM forwarding).

🚀 Step-by-Step Implementation
1. Setting up the PPG Gateway (Logstash)
Installed Logstash on an Ubuntu intermediary server and configured the pipeline (/etc/logstash/conf.d/01-ppg-filter.conf) to intercept logs and apply data masking.

The Redaction Rules:

Ruby
filter {
  # Rule 1: Redact Email Addresses
  mutate {
    gsub => [ "message", "[\w\.-]+@[\w\.-]+\.\w+", "[PII_REDACTED]" ]
  }

  # Rule 2: IP Anonymization (Keep network context, hide specific host)
  mutate {
    gsub => [ "message", "(\d+\.\d+\.\d+)\.\d+", "\1.0" ]
  }
}
2. Configuring SIEM Ingestion (Wazuh)
Configured Logstash to forward the sanitized logs to the Wazuh Manager using standard Syslog.

Ruby
output {
  udp {
    host => "192.168.x.x" # Wazuh Manager IP
    port => 514
    codec => plain { format => "%{message}" }
  }
}
3. Creating Custom Wazuh Detection Rules
Because the raw log format was customized, a local rule was required in Wazuh (/var/ossec/etc/rules/local_rules.xml) to trigger an alert when the sanitized failed login attempt arrived.

XML
<group name="local,syslog,">
  <rule id="100050" level="7">
    <match>failed login from</match>
    <description>PPG Lab Alert: Failed login attempt (PII Sanitized)</description>
  </rule>
</group>
🛠️ Challenges & Troubleshooting
Building this pipeline required resolving several configuration and communication errors between the systems.

❌ Issue 1: Netcat syntax compatibility
Error: nc: unrecognized option: -t when attempting to send test JSON logs from the source machine.

Root Cause: The OpenBSD version of Netcat installed on the source system does not support the legacy -t (Telnet negotiation) flag.

Resolution: Removed the flag and piped the string directly: echo '{"message": "..."}' | nc <IP> 5044.

❌ Issue 2: Logstash Configuration Crash
Error: LogStash::ConfigurationError: Expected one of [0-9]... after output { udp { host => 192.168

Root Cause: In the Logstash output block, the Wazuh IP address was entered as an integer/bareword rather than a string.

Resolution: Enclosed the IP address in double quotes ("192.168.x.x") and validated the configuration using /usr/share/logstash/bin/logstash --config.test_and_exit.

❌ Issue 3: Logs processed by Logstash but dropped by Wazuh
Error: Logstash successfully received and sanitized the logs (verified via stdout rubydebug), and threw minor ECS compatibility warnings, but no alerts appeared in the Wazuh Dashboard.

Root Cause: 1. Logstash was sending the entire JSON object (including @timestamp and metadata) which Wazuh's default Syslog decoder didn't parse correctly.
2. Wazuh did not have a matching rule for the custom string.
3. Wazuh Manager was not actively listening on UDP 514 for remote syslogs.

Resolution: 1. Modified the Logstash output codec to plain { format => "%{message}" } to send only the sanitized text.
2. Created custom rule 100050 and verified it using /var/ossec/bin/wazuh-logtest.
3. Added the <remote> syslog configuration block in /var/ossec/etc/ossec.conf to explicitly open port 514 and allow the Logstash IP.

🏁 Final Results
By sending a simulated attack log containing sensitive PII:
Alert: User sarah_admin@bank.iq failed login from 10.0.5.22

The Wazuh Dashboard successfully generated a Level 7 alert displaying only the sanitized data, fulfilling the privacy requirements:
Alert: User [PII_REDACTED] failed login from 10.0.5.0
