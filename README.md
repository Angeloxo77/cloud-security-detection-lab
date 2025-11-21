📁 Security Hardening & Detection Pipeline

Author: Ángel García
Role: Security Engineer / Blue Team / Cloud Security
Target companies: Microsoft, Google, Amazon, Apple

🧠 Overview

This project builds a complete security monitoring and hardening environment using cloud infrastructure, endpoint protection, SIEM integrations, and adversary simulation based on MITRE ATT&CK.

The goal is to demonstrate real-world security engineering skills, including defensive configuration, attack detection, cloud automation, and incident response.

🏗️ Architecture

(Insertar diagrama aquí)

Components:

Azure (network, VMs, Sentinel)

Windows Server (CIS hardened)

Ubuntu Linux (CIS hardened)

Sysmon + Azure Agent

Kali/Parrot attacker

SOAR automation

🔧 Project Structure
/infra/             → Terraform IaC  
/hardening/         → Windows & Linux CIS scripts  
/detections/        → SIEM rules (KQL)  
/automation/        → SOAR Playbooks  
/attacks/           → Red Team simulation scripts  
/report/            → Final audit report  

🛡️ Hardening Highlights
Windows:

CIS Benchmark

Sysmon

ASR Rules

LAPS

PowerShell logging

Defender config

Linux:

SSH hardening

Fail2Ban

UFW firewall

AppArmor

Sysctl security config

👁️ Detection Engineering

Created 15+ SIEM rules including:

Mimikatz detection

LSASS access

SSH brute force

Sudo privilege escalation

File integrity alerts

Process injection patterns

⚔️ Attack Simulations

Simulated techniques:

Reconnaissance

Brute force

Privilege escalation

Lateral movement

Persistence

Exfiltration

All detections were validated through live attacks.

🚨 SOAR Automation

Playbooks:

Block attacker IP

Alert via Slack/Telegram

Auto-tagging incidents

Gather forensic logs

📘 Final Report

A professional audit-style report including:

Diagram

Threat model

Controls

Detections

Attack evidence

Recommendations