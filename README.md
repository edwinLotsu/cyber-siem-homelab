# Cybersecurity Home Lab: SIEM Deployment & Threat Detection

A hands-on security operations lab demonstrating SIEM deployment, endpoint monitoring, attack simulation, and detection engineering using Wazuh, Active Directory, and Kali Linux.

---

## Objectives

- Deploy enterprise-grade SIEM infrastructure (Wazuh)
- Monitor Windows endpoints with Sysmon telemetry
- Simulate real-world attack scenarios
- Develop custom detection rules
- Map detections to MITRE ATT&CK framework

---

## Lab Architecture

### Components
- **Windows Server 2022** → Active Directory Domain Controller (`lab.local`)
- **Windows 10 Enterprise** → Domain-joined client monitored endpoint
- **Kali Linux** → Offensive security platform (attacker simulation)
- **Ubuntu Server 22.04** → Wazuh SIEM infrastructure (Manager + Indexer + Dashboard)

### Networking
- **Host-Only Network** (`192.168.86.0/24`) → Isolated lab traffic
- **NAT Network** → Internet access for updates/tools

### IP Addressing
| System | IP Address | Role |
|--------|------------|------|
| Windows Server 2022 | 192.168.86.128 | Domain Controller |
| Windows 10 Client | 192.168.86.129 | Monitored Endpoint |
| Kali Linux | 192.168.86.131 | Attacker Machine |
| Ubuntu SIEM | 192.168.86.130 | Wazuh Infrastructure |

---

## Projects Completed

### 1. Wazuh SIEM Deployment
**What:** Deployed multi-component SIEM infrastructure on Ubuntu Server  
**Why:** Centralized log collection, correlation, and alerting for security monitoring  
**[View Project →](01-wazuh-siem-deployment/)**

**Key Achievements:**
- Wazuh Manager installed and configured
- Elasticsearch-based indexer for log storage
- Web-based dashboard for visualization
- Agents deployed to 2 Windows endpoints (100% active status)

---

### 2. Sysmon Endpoint Telemetry
**What:** Enhanced Windows endpoint visibility with Sysmon process/network monitoring  
**Why:** Windows Event Logs alone miss critical security events — Sysmon fills the gap  
**[View Project →](02-sysmon-deployment/)**

**Key Achievements:**
- Sysmon deployed on Windows Server + Windows 10
- Network connection logging enabled (Event ID 3)
- Process creation monitoring (Event ID 1)
- Configuration validated with test scenarios

---

### 3. Attack Simulation: Nmap Port Scanning
**What:** Simulated network reconnaissance attack from Kali against Windows 10  
**Why:** Validate SIEM can detect port scanning — a common reconnaissance technique  
**[View Project →](03-attack-simulations/01-nmap-port-scan/)**

**Attack Details:**
```bash
nmap -sT -4 -p 80,135,139,445,3389 192.168.86.129
```

**Detection:**
- 9 Sysmon Event ID 3 logs captured in ~1.5 seconds
- Wazuh timeline showed clear spike during scan window
- Mapped to MITRE ATT&CK T1046 (Network Service Scanning)

---

### 4. Attack Simulation: Failed Login Detection
**What:** Generated failed authentication attempts to test brute force detection  
**Why:** Credential attacks are a top threat — verify logging and alerting works  
**[View Project →](03-attack-simulations/02-failed-login-detection/)**

**Attack Details:**
```cmd
runas /user:lab\Administrator cmd
```

**Detection:**
- Windows Event ID 4625 (Failed Login) captured
- Wazuh correlated multiple failures
- Can be used to build threshold-based brute force alerts

---

## Tools & Technologies

| Category | Tools |
|----------|-------|
| **SIEM** | Wazuh (Manager, Indexer, Dashboard) |
| **Endpoint Monitoring** | Sysmon, Windows Event Logs |
| **Infrastructure** | Active Directory, Windows Server 2022, VMware |
| **Offensive Tools** | Kali Linux, Nmap |
| **Operating Systems** | Ubuntu Server, Windows 10, Windows Server 2022 |

---

## Skills Demonstrated

- SIEM deployment and configuration
- Log collection and centralized monitoring
- Endpoint telemetry engineering (Sysmon)
- Attack simulation and red team techniques
- Blue team detection and response
- MITRE ATT&CK framework mapping
- Windows Active Directory administration
- Linux system administration
- Network security monitoring

---

## Next Steps

- [ ] Brute force attack simulation (Hydra against RDP)
- [ ] PowerShell attack detection (Script Block Logging)
- [ ] Network traffic capture and correlation (Wireshark + SIEM)
- [ ] Custom Wazuh detection rules
- [ ] Lateral movement simulation

---

## Documentation Structure

Each project includes:
- **README.md** → Overview and objectives
- **Screenshots/** → Visual proof of work
- **Configs/** → Sanitized configuration files
- **Analysis.md** → Lessons learned and detection methodology

---

## Connect

**LinkedIn:** [Edwin Lotsu](https://www.linkedin.com/in/lotsuedwin)  
**Resume:** [View PDF](#)

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

**Note:** This is a controlled lab environment for educational purposes. All attack simulations are performed on isolated virtual machines with no connection to production systems.
