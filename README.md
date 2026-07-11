# Seclab Sensor — Docker Network Monitoring Lab

## Legal Notice
This project was built and tested entirely inside an isolated Docker network (`seclab-net`) on a machine I own and control. It includes DVWA (Damn Vulnerable Web Application), a tool intentionally built with security flaws for training purposes.

This setup, including any default credentials used, is intended strictly for an isolated lab environment. It should never be replicated on a production system, an internet-facing host, or any network you do not own or have explicit authorization to test on.

## Objective
Design and deploy a lightweight HTTP logging service (`seclab-sensor`) inside an isolated Docker network, and validate inter-container communication and basic request monitoring across four containers: the custom Flask logger, a plain Ubuntu host, DVWA, and a Kali Linux tools container.

## Repository Structure
```
Seclab-Sensor/
├── README.md         → Overview, setup, and how to reproduce the lab
├── report.md          → Full technical breakdown, findings, and risk analysis
├── app.py              → The Flask logging application
├── Dockerfile           → Build recipe for the seclab-sensor container
└── screenshots/          → Evidence for each step, referenced in report.md
```

## Tools Used
- Docker
- Flask (Python) — custom logging application
- DVWA (Damn Vulnerable Web Application)
- Kali Linux (container + host)
- Ubuntu (container)
- curl

## Setup
1. Create the isolated Docker network:
   ```
   docker network create seclab-net
   ```
2. Build the logger image:
   ```
   docker build -t seclab-sensor .
   ```
3. Start the logging container:
   ```
   docker run -d --name seclab-sensor --network seclab-net -p 5000:5000 seclab-sensor
   ```
4. Start the remaining containers on the same network:
   ```
   docker run -dit --name kali-linux --network seclab-net kalilinux/kali-rolling bash
   docker run -dit --name ubuntu --network seclab-net ubuntu bash
   docker run -d --name dvwa --network seclab-net -p 8080:80 vulnerables/web-dvwa
   ```
5. Confirm all four containers are running:
   ```
   docker ps
   ```
6. Test inter-container communication and view captured logs — see `report.md` for the full walkthrough and findings.
