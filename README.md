# WAF + Wazuh Web Attack Detection

A practical cybersecurity lab demonstrating how a **Web Application Firewall (WAF)** can detect and block web attacks and how security events can be forwarded to **Wazuh SIEM** for centralized monitoring and analysis.

The project uses **Apache, ModSecurity, OWASP Core Rule Set (CRS), and Wazuh** to create a web attack detection and monitoring pipeline.

---

## Architecture

```text
                         ATTACKER
                            │
                            │ HTTP/HTTPS
                            ▼
                  ┌───────────────────┐
                  │    Web Server     │
                  │      Apache       │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   ModSecurity     │
                  │       WAF         │
                  │                   │
                  │   OWASP CRS       │
                  └─────────┬─────────┘
                            │
                    Detect / Block
                            │
                            ▼
                  ┌───────────────────┐
                  │   Apache / WAF    │
                  │       Logs        │
                  └─────────┬─────────┘
                            │
                            │ Log collection
                            ▼
                  ┌───────────────────┐
                  │   Wazuh Agent     │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   Wazuh Manager   │
                  │       SIEM        │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │  Wazuh Dashboard  │
                  │                   │
                  │ Security Alerts   │
                  │ Logs / Analysis   │
                  └───────────────────┘
```

---

# 1. Project Objectives

The main objectives of this project are:

* Deploy an Apache web server.
* Configure ModSecurity as a Web Application Firewall.
* Deploy OWASP Core Rule Set (CRS).
* Detect common web attacks.
* Block malicious HTTP requests.
* Collect ModSecurity and Apache logs with Wazuh.
* Monitor web attacks through the Wazuh dashboard.
* Create custom Wazuh rules when additional detection logic is required.
* Demonstrate the complete attack-to-alert workflow.

---

# 2. Technologies

| Technology      | Purpose                    |
| --------------- | -------------------------- |
| Apache          | Web server                 |
| ModSecurity     | Web Application Firewall   |
| OWASP CRS       | Web attack detection rules |
| Wazuh Agent     | Log collection             |
| Wazuh Manager   | SIEM and correlation       |
| Wazuh Dashboard | Security monitoring        |
| Linux/Ubuntu    | Server operating system    |

---

# 3. How the System Works

The system follows this process:

```text
Web Request
     │
     ▼
ModSecurity
     │
     ├── Normal request ──► Apache
     │
     └── Malicious request
              │
              ▼
         OWASP CRS
              │
              ▼
        Block / Detect
              │
              ▼
          WAF Log
              │
              ▼
        Wazuh Agent
              │
              ▼
        Wazuh Manager
              │
              ▼
       Security Alert
```

ModSecurity analyzes incoming HTTP requests.

OWASP CRS provides detection rules for common web attacks.

When a malicious request matches a rule, ModSecurity can generate a security event and, depending on the configuration, reject the request.

Wazuh collects the resulting logs and presents the events in the Wazuh Dashboard.

---

# 4. Installation

## 4.1 Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

## 4.2 Install Apache

```bash
sudo apt install apache2 -y
```

Check the service:

```bash
sudo systemctl status apache2
```

Test the web server:

```bash
curl http://localhost
```

---

# 5. ModSecurity Installation

Install ModSecurity for Apache:

```bash
sudo apt install libapache2-mod-security2 -y
```

Enable the module:

```bash
sudo a2enmod security2
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

Verify that ModSecurity is loaded:

```bash
apache2ctl -M | grep security
```

Expected output should contain:

```text
security2_module
```

---

# 6. Enable ModSecurity

Copy the recommended configuration:

```bash
sudo cp /etc/modsecurity/modsecurity.conf-recommended \
/etc/modsecurity/modsecurity.conf
```

Edit the configuration:

```bash
sudo nano /etc/modsecurity/modsecurity.conf
```

Change:

```text
SecRuleEngine DetectionOnly
```

to:

```text
SecRuleEngine On
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

---

# 7. Install OWASP Core Rule Set

Install the CRS package:

```bash
sudo apt install modsecurity-crs -y
```

CRS provides detection rules for many classes of web attacks, including:

* SQL Injection
* Cross-Site Scripting
* Local File Inclusion
* Remote File Inclusion
* Command Injection
* Path Traversal
* Protocol violations
* Scanner activity
* Other malicious web requests

---

# 8. Verify ModSecurity

Check Apache configuration:

```bash
sudo apache2ctl configtest
```

Expected:

```text
Syntax OK
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

Check ModSecurity logs:

```bash
sudo tail -f /var/log/apache2/error.log
```

Depending on the configuration, ModSecurity audit events can also be found in:

```bash
/var/log/modsec_audit.log
```

---

# 9. Web Attack Testing

The attacks in this project should only be performed against systems that you own or are authorized to test.

## SQL Injection

Example test request:

```bash
curl "http://SERVER_IP/?id=1%27%20OR%201%3D1"
```

A properly configured WAF may detect and reject the request.

---

## Cross-Site Scripting

Example:

```bash
curl "http://SERVER_IP/?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E"
```

ModSecurity/CRS can identify suspicious XSS patterns.

---

## Directory Traversal

Example:

```bash
curl "http://SERVER_IP/?file=../../../../etc/passwd"
```

The request can trigger a path-traversal detection rule.

---

## Command Injection

Example:

```bash
curl "http://SERVER_IP/?host=127.0.0.1%3Bwhoami"
```

This can be used to test command-injection detection in an intentionally vulnerable lab application.

---

## Suspicious File Access

Examples:

```bash
curl "http://SERVER_IP/.env"
```

```bash
curl "http://SERVER_IP/phpmyadmin"
```

```bash
curl "http://SERVER_IP/admin"
```

These requests can be used to demonstrate reconnaissance and sensitive-resource access monitoring.

---

# 10. Wazuh Integration

The Wazuh Agent is installed on the web server.

Its purpose is to collect Apache and ModSecurity logs and forward them to the Wazuh Manager.

Edit:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the appropriate log locations.

For example:

```xml
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/access.log</location>
</localfile>

<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/error.log</location>
</localfile>

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/modsec_audit.log</location>
</localfile>
```

The exact `log_format` should match the log format and Wazuh decoder available in your environment.

Restart the agent:

```bash
sudo systemctl restart wazuh-agent
```

Check:

```bash
sudo systemctl status wazuh-agent
```

---

# 11. Wazuh Custom Rules

Wazuh custom rules can be stored in:

```bash
/var/ossec/etc/rules/local_rules.xml
```

Example:

```xml
<group name="web_attack,modsecurity,">

  <rule id="100200" level="10">
    <match>ModSecurity: Access denied</match>
    <description>Web Attack: ModSecurity blocked malicious request</description>
    <group>web_attack,modsecurity,</group>
  </rule>

</group>
```

This rule can generate a Wazuh alert when the corresponding ModSecurity event appears in the collected logs.

The exact rule should be adapted to the actual event format received by Wazuh.

---

# 12. Detection Flow

Example SQL Injection event:

```text
Attacker
   │
   │ SQL Injection request
   ▼
ModSecurity
   │
   ▼
OWASP CRS
   │
   │ Attack detected
   ▼
Request blocked
   │
   ▼
ModSecurity Log
   │
   ▼
Wazuh Agent
   │
   ▼
Wazuh Manager
   │
   ▼
Wazuh Rule
   │
   ▼
🚨 Security Alert
   │
   ▼
Wazuh Dashboard
```

---

# 13. Monitoring in Wazuh

After generating test traffic, investigate the events in:

**Wazuh Dashboard → Security Events**

Useful fields to examine include:

* Source IP
* Destination IP
* HTTP method
* Requested URI
* HTTP status
* Rule ID
* Alert level
* ModSecurity message
* User-Agent
* Timestamp

Example investigation:

```text
Source IP:
192.168.1.100

Request:
/index.php?id=...

Detection:
SQL Injection

WAF:
ModSecurity

Action:
Blocked

SIEM:
Wazuh
```

---

# 14. Attack Categories

The project can demonstrate the following categories:

| Attack                | Detection Layer           |
| --------------------- | ------------------------- |
| SQL Injection         | ModSecurity / OWASP CRS   |
| XSS                   | ModSecurity / OWASP CRS   |
| Command Injection     | ModSecurity / OWASP CRS   |
| Path Traversal        | ModSecurity / OWASP CRS   |
| File Inclusion        | ModSecurity / OWASP CRS   |
| Scanner activity      | ModSecurity / Apache logs |
| Suspicious paths      | Apache logs / Wazuh rules |
| Sensitive file access | Apache logs / Wazuh rules |

---

# 15. Why Use Both WAF and Wazuh?

The WAF and SIEM have different responsibilities.

### ModSecurity

ModSecurity operates close to the web application and analyzes HTTP requests.

Its primary role is:

```text
Detect → Inspect → Block
```

### Wazuh

Wazuh provides centralized security monitoring.

Its role is:

```text
Collect → Analyze → Correlate → Alert → Investigate
```

Therefore:

```text
ModSecurity = Web protection

Wazuh = Security monitoring
```

They complement each other rather than replacing each other.

---

# 16. Project Results

The final system demonstrates that:

1. A web request reaches the server.
2. ModSecurity inspects the request.
3. OWASP CRS evaluates the request against security rules.
4. Malicious requests can be rejected.
5. ModSecurity generates security logs.
6. Wazuh Agent collects the logs.
7. Wazuh Manager processes the events.
8. Security alerts appear in the Wazuh Dashboard.
9. The analyst can investigate the attack details.

---

# 17. Screenshots

Recommended screenshots for this repository:

```text
screenshots/
├── apache-running.png
├── modsecurity-enabled.png
├── owasp-crs.png
├── sql-injection-blocked.png
├── xss-blocked.png
├── modsecurity-log.png
├── wazuh-agent.png
├── wazuh-alert.png
└── wazuh-dashboard.png
```

Screenshots should show the actual environment and results from your lab.

---

# 18. Future Improvements

Possible extensions include:

* Teler for additional web-log analysis
* Suricata IDS/IPS
* Automated response
* Shuffle SOAR integration
* Telegram/email notifications
* Threat-intelligence enrichment
* Custom Wazuh correlation rules
* Attack-source IP blocking
* Centralized dashboards
* Detection engineering based on MITRE ATT&CK

---

# 19. Conclusion

This project demonstrates a practical security monitoring architecture in which **ModSecurity provides web application protection while Wazuh provides centralized security monitoring and alerting**.

The combination creates a useful defensive pipeline:

```text
        WEB ATTACK
             │
             ▼
       ModSecurity
             │
       OWASP CRS
             │
      Detect / Block
             │
             ▼
        Security Log
             │
             ▼
        Wazuh Agent
             │
             ▼
       Wazuh Manager
             │
             ▼
      Wazuh Dashboard
             │
             ▼
       SOC Investigation
```

> **Purpose:** This project is intended for authorized cybersecurity training, defensive security research, and laboratory environments.
