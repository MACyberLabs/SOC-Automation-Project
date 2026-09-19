# SOC-Automation-Project
This project implements an automated Security Operations Center (SOC) environment for monitoring security events, detecting suspicious activity, enriching alerts with threat intelligence, and managing incidents through a centralized case management platform.
The lab integrates Wazuh, Sysmon, Shuffle, VirusTotal, and TheHive to create an automated security monitoring and incident response pipeline.
The primary goal is to demonstrate how security events can move from endpoint detection to automated investigation and incident management, reducing repetitive manual tasks and providing security analysts with the information required to investigate potential threats.

Technologies:
Wazuh — Security monitoring, log analysis, and threat detection
Sysmon — Windows endpoint telemetry
Shuffle — Security orchestration and automation
VirusTotal — Threat intelligence and IOC enrichment
TheHive — Incident response and case management
Windows — Monitored endpoint

Architecture:
![image alt](https://github.com/MohamedAbdallaProjects/SOC-Automation-Project/blob/main/Image%20(1).png?raw=true)

Endpoint Monitoring:
The Windows endpoint is monitored using Sysmon, which provides detailed information about activity occurring on the system.
Sysmon telemetry can provide visibility into:
Process creation, Parent-child process relationships, Command line execution, Network connections, File activity, Executable execution, Process behavior, Other security-relevant system events.
The collected telemetry is forwarded to Wazuh for centralized analysis.

Wazuh:
Wazuh serves as the primary security monitoring and detection platform.
The Wazuh agent collects endpoint events and forwards them to the Wazuh manager, where they are analyzed using detection rules.
Wazuh is responsible for:
Endpoint monitoring, Log collection, Event analysis, Security detection, Custom detection rules, Alert generation, File integrity monitoring, Windows event monitoring.
When suspicious activity matches configured detection logic, Wazuh generates a security alert that can be passed to the automation workflow.

Shuffle Automation:
Shuffle provides the SOAR layer of the environment.
Selected Wazuh alerts are forwarded to Shuffle through a webhook. Shuffle processes the alert and executes a predefined workflow.
The automation process can:
Receive the Wazuh alert, Parse the alert data, Extract relevant indicators, Perform threat intelligence lookups, Process the returned information, Create an alert or case in TheHive, Notify the analyst of the incident.
This allows repetitive investigation tasks to be performed automatically.

Threat Intelligence:
The automation workflow extracts Indicators of Compromise (IOCs) from security alerts.
Common indicators include:
File hashes, IP addresses, Domains, URLs, File names, Hostnames, User information.
These indicators can be submitted to VirusTotal to obtain additional reputation and threat intelligence information.

TheHive
TheHive provides the incident response and case management component of the project.
After the alert has been processed and enriched, relevant information is forwarded to TheHive.
A case can contain information such as: 
Incident description, Detection timestamp, Affected host, Username, Process information, File hashes, IP addresses, Detection rules, Threat intelligence results, Severity, Investigation notes.
TheHive provides a centralized workspace for tracking incidents, organizing observables, documenting investigations, and managing the incident response process.

Detection and Investigation:
The environment can be used to simulate suspicious activity on the monitored endpoint and validate the complete detection pipeline.
Detection scenario follows:
Suspicious Activity -> Sysmon -> Wazuh agent -> Wazuh Manager -> Detection Rule -> Wazuh Alert -> Shuffle -> IOC Extraction -> VirusTotal Lookup -> TheHive -> Analyst Investigation.
This demonstrates how endpoint activity can be transformed into a structured incident requiring analyst investigation.

Incident Response Lifecycle:
The project follows a complete security alert lifecycle:
Detection:
Endpoint telemetry is collected and analyzed by Wazuh.
Alerting:
Wazuh generates an alert when configured detection criteria are met.
Automation:
Shuffle receives the alert and executes the predefined investigation workflow.
Enrichment:
Relevant indicators are analyzed using threat intelligence services.
Case Management:
The processed alert is sent to TheHive for incident tracking.
Investigation:
The analyst reviews the alert, observables, evidence, and enrichment results.
Response:
Appropriate response actions can then be determined and documented.

Technologies Used:
Windows:
Monitored endpoint
Sysmon:
Endpoint telemetry
Wazuh:
Security monitoring and detection
Shuffle:
SOAR and workflow automation
VirusTotal:
Threat intelligence enrichment
TheHive:
Incident response and case management
Webhooks:
Alert integration
APIs:
Platform communication

Project Objectives:
Implement centralized endpoint monitoring.
Collect detailed Windows security telemetry.
Detect suspicious activity using Wazuh.
Develop and test security detection rules.
Automate alert processing with Shuffle.
Extract and investigate Indicators of Compromise.
Enrich security alerts with threat intelligence.
Automatically create incidents in TheHive.
Streamline the SOC investigation workflow.
Demonstrate integration between SIEM, SOAR, threat intelligence, and incident response platforms.

Key Concepts Demonstrated:
SIEM / Security Monitoring
Endpoint Detection
Detection Engineering
Log Analysis
SOC Alert Triage
SOAR Automation
Threat Intelligence
IOC Analysis
Incident Response
Case Management
API Integration
Webhook Automation

Future Improvements:
Potential extensions to the environment include:
Additional Windows and Linux endpoints
Expanded Wazuh detection rules
MITRE ATT&CK mapping
MISP integration
Cortex integration
Additional threat intelligence sources
Automated IOC correlation
Automated containment actions
Automated IP blocking
Slack or Microsoft Teams notifications
Additional Shuffle playbooks
SOC metrics and dashboards
Additional attack simulations
Automated incident response playbooks
