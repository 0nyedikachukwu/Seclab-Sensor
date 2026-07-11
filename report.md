# Seclab Sensor — Technical Report

## Executive Summary
This report documents the design and deployment of a custom Flask-based HTTP logging service running inside an isolated Docker network, alongside three other containers (Ubuntu, DVWA, and Kali Linux) used to generate and validate traffic. It covers the build process, inter-container communication testing, and a notable design detail uncovered during testing: the sensor returns deliberately misleading responses to visitors while logging every request in the background.

## Project Overview

### Purpose
Demonstrate how to isolate and monitor traffic between containers on a private Docker network, and build a lightweight custom sensor capable of logging visits and flagging suspicious request paths.

### Key Steps
1. Create an isolated Docker network
2. Build and deploy the custom Flask logging container
3. Deploy supporting containers (Ubuntu, DVWA, Kali Linux)
4. Generate test traffic from inside and outside the Docker network
5. Review captured logs

## Technical Implementation

### 1. Network Creation
The lab network was created with:
```
docker network create seclab-net
```
![Network creation](screenshots/01_network_create.png)
*Creating the isolated Docker network.*

This was confirmed with:
```
docker network ls
```
![Network list](screenshots/02_network_ls.png)
*`seclab-net` listed alongside Docker's default networks.*

This creates a private Docker bridge network where containers can resolve each other by container name instead of IP address — the foundation for the inter-container communication tested later.

### 2. Logging Application (`app.py`)
Written with `nano app.py`. The application listens for incoming requests, logs each visit (timestamp, method, path, source IP, and user agent), and checks incoming paths against a list of suspicious-looking names (`/admin`, `/.env`, `/wp-login.php`, `/login`, `/config`, `/.git`).

![app.py source](screenshots/20_app_py_code.png)
*Full source of the Flask logging application.*

A closer look at the code reveals a deliberate design choice: the `/` and `/log` routes don't behave like normal endpoints. Every request is logged silently in the background, but the routes always return a fixed, misleading response regardless of what actually happened — `/` always returns a fake "504 Gateway Timeout," and `/log` always returns a fake "401 Incorrect username or password." In other words, this sensor behaves like a lightweight honeypot: it collects data on every visitor while showing them a convincing but false error, rather than confirming it logged anything.

### 3. Build Recipe (`Dockerfile`)
Written with `nano Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install flask
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```
![Dockerfile content](screenshots/05_dockerfile_content.png)
*Build recipe packaging the Flask app into a container image.*

### 4. Building and Running the Containers
The image was built with:
```
docker build -t seclab-sensor .
```
![Docker build](screenshots/06_docker_build.png)

The logging container was started with:
```
docker run -d --name seclab-sensor --network seclab-net -p 5000:5000 seclab-sensor
```
![Starting seclab-sensor](screenshots/07_run_seclab_sensor.png)
*Container ID redacted.*

The remaining three containers were started on the same network:
```
docker start kali-linux
docker start ubuntu
docker start dvwa
```
![Starting Kali](screenshots/08_start_kali.png)
![Starting Ubuntu](screenshots/09_start_ubuntu.png)
![Starting DVWA](screenshots/10_start_dvwa.png)

All four containers were confirmed running with:
```
docker ps
```
![docker ps output](screenshots/11_docker_ps.png)
*Container IDs redacted; port bindings (5000, 8080) visible.*

### 5. Testing Communication from the Ubuntu Container
Entered the Ubuntu container and installed a tool for sending test requests:
```
docker exec -it ubuntu bash
apt update && apt install curl -y
```
![Installing curl in Ubuntu](screenshots/12_ubuntu_install_curl.png)

A request to DVWA using its container name resolved and returned the DVWA login page HTML:
```
curl http://dvwa:80
```
![Curl to DVWA from Ubuntu](screenshots/13_ubuntu_curl_dvwa.png)

Requests to the sensor returned its decoy responses rather than real data:
```
curl http://seclab-sensor:5000/
curl http://seclab-sensor:5000/log
```
![Curl to sensor from Ubuntu](screenshots/14_ubuntu_curl_sensor.png)
*`/` returned a fake 504; `/log` returned a fake 401 — both requests were still logged in the background (confirmed in Section 7).*

The session was then closed with `exit`.

### 6. Testing Communication from Kali (Container vs. Host)
From inside the `kali-linux` container, the same name-based addressing worked as expected, producing the same decoy responses seen from Ubuntu:
```
curl http://seclab-sensor:5000/
curl http://seclab-sensor:5000/log
```
![Curl to sensor from Kali container](screenshots/15_kali_container_curl_sensor.png)

```
curl -L http://dvwa:80
```
![Curl to DVWA from Kali container](screenshots/16_kali_container_curl_dvwa.png)

From the Kali machine itself — outside the Docker network — container names could not be resolved, since name-based DNS only works between containers on the same Docker network. Instead, `localhost` with the published port was used:
```
curl http://localhost:5000/
curl http://localhost:5000/log
curl http://localhost:5000/admin
```
![Curl to sensor from Kali host](screenshots/17_kali_host_curl_sensor.png)
*`/admin` isn't a defined route, so Flask returned its own default 404 — but the request was still captured by the logger, since the logging hook runs before route handling.*

This distinction is a key finding: **containers talk to each other by name; anything outside the Docker network has to use the host's published ports instead.**

### 7. DVWA Web Access
From the Kali host, DVWA was accessed directly in a browser at `http://localhost:8080`, and logged into using DVWA's default credentials (`admin` / `password`).

![DVWA setup page](screenshots/18_dvwa_browser_setup.png)
*DVWA's database setup screen, confirming the application was reachable and running.*

*Default credentials are intentional here — DVWA is designed to be vulnerable for training. This should never be replicated on a real or internet-facing system.*

### 8. Reviewing Captured Logs
The sensor's captured activity was reviewed with:
```
docker logs seclab-sensor
```
![Captured logs](screenshots/19_docker_logs_seclab_sensor.png)
*Internal container IPs redacted; status codes and paths visible.*

This confirmed every test request — from Ubuntu, from the Kali container, and from the Kali host — was recorded, including the `/admin` request that received a 404 on the surface but was still logged and flagged as a hit against the suspicious-path list.

## Findings
- Container-to-container requests resolved successfully using Docker's built-in DNS (container names), confirming the custom network was functioning correctly.
- Host-to-container requests required `localhost` + published port, since container names are not resolvable outside the Docker network.
- The sensor's `/` and `/log` endpoints do not return real status information — they return fixed decoy responses (fake 504, fake 401) regardless of the actual request, while silently logging every visit in the background. This wasn't apparent from the endpoint responses alone; it only became clear from reading `app.py` directly.
- The `/admin` path (in the `SUSPICIOUS_PATHS` list) has no defined route, so it fell through to Flask's default 404 — but was still captured by the global `before_request` logging hook, confirming suspicious-path requests are logged even when no explicit handler exists for them.
- `docker logs` confirmed all test traffic was recorded with timestamp, method, path, and source container IP, across all three traffic sources.

## Risks & Mitigation

| Risk Observed | Why It Matters | Mitigation |
|---|---|---|
| DVWA deployed with default credentials (`admin`/`password`) | Trivial to compromise if ever exposed beyond the isolated lab | Never reuse default credentials outside a lab; keep DVWA off any shared or internet-facing network |
| Flask logger exposed on port 5000 with no authentication | Anyone able to reach the container could send requests and be logged, or potentially discover the decoy behavior through repeated probing | Add authentication to sensitive endpoints, or restrict access to the lab network only |
| All containers share one flat network with no segmentation | A compromised container (e.g. DVWA) could freely reach the others | Apply network segmentation or firewall rules between services in any non-lab deployment |
| Sensor's decoy responses are undocumented at the endpoint level | Anyone relying on the HTTP response alone (rather than the logs) would draw the wrong conclusion about what the service does | Document decoy/honeypot behavior clearly for anyone operating or auditing the service |

## Conclusion
This lab validated core Docker networking concepts: containers on a custom network can resolve each other by name, while external hosts must rely on published ports instead. The custom Flask sensor successfully captured and logged traffic from multiple sources, and turned out to function as a basic honeypot — presenting misleading decoy responses to visitors while recording every request in the background. The main takeaways are the clear operational difference between internal (name-based) and external (port-based) addressing, and the importance of reading a service's actual source code rather than trusting its HTTP responses at face value.
