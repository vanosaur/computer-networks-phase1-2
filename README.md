# Private Network Service Platform
## Computer Networks — Phase 1 & Phase 2

A team-based local networking project demonstrating DNS, TCP/IP, HTTP, HTTPS/TLS, reverse proxying, load balancing, HTTP caching, packet analysis, backend failure handling, DNS redundancy, firewall-based service isolation, high availability, and controlled edge migration in a private LAN environment.

---

## Team Members

| Name | Role | IP Address | Services |
|---|---|---|---|
| Vani Rudra | Primary DNS + Backend A | `10.7.9.62` | dnsmasq, Backend A |
| Apoorva Choudhary | Primary Nginx Edge | `10.7.31.1` | Nginx, HTTPS, Load Balancer |
| Vidhi | Backend B + Backup DNS + Standby Edge | `10.7.7.85` | Backend B, dnsmasq, Nginx |

---

## Network Configuration

| Parameter | Value |
|---|---|
| Network | `10.7.0.0/19` |
| Subnet Mask | `255.255.224.0` |
| Default Gateway | `10.7.0.1` |
| Primary DNS Server | `10.7.9.62` |
| Backup DNS Server | `10.7.7.85` |

---

## Project Architecture

```text
                         Private LAN
                         10.7.0.0/19
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
       Mac 1               Mac 2               Mac 3
       Vani               Apoorva                Vidhi
     10.7.9.62           10.7.31.1            10.7.7.85
          |                   |                   |
     Primary DNS          Primary Edge        Backup DNS
       :53                 Nginx              Backend B
     Backend A            HTTP :8080           :3002
       :3001              HTTPS :8443         Standby Nginx
          |                   |                   |
          +-------------------+-------------------+
                              |
                       Backend A / B
                       Load Balancing
```

## Project Overview

The project consists of three macOS laptops connected to the same private LAN.

Each machine has a specific role:

- Mac 1 provides private DNS using dnsmasq and hosts Backend A.
- Mac 2 acts as the primary edge server using Nginx, HTTPS, reverse proxying, and load balancing.
- Mac 3 hosts Backend B, acts as the backup DNS resolver, and provides a standby Nginx edge server.
- All three machines communicate using private IPv4 addresses.

The project demonstrates how a client request travels through DNS resolution, TCP connectivity, TLS-secured HTTPS, Nginx reverse proxying, load balancing, backend processing, and response delivery.

Phase 2 extends the original implementation by introducing DNS redundancy, TTL management, firewall-based service isolation, backend failover, controlled edge migration, and network troubleshooting.

---

# PHASE 1 — CORE NETWORKING IMPLEMENTATION

## Project Components

### Mac 1 — Primary DNS + Backend A

**Team Member:** Vani Rudra  
**IP Address:** `10.7.9.62`

**Services:**

- `dnsmasq` → Port 53
- Backend A → Port 3001

**Responsibilities:**

- Primary private DNS server
- Backend A hosting
- DNS resolution for project domains
- Client-side testing

### Mac 2 — Nginx + HTTPS + Load Balancer

**Team Member:** Apoorva Choudhary  
**IP Address:** `10.7.31.1`

**Services:**

- Nginx HTTP → Port 8080
- Nginx HTTPS → Port 8443

**Responsibilities:**

- Primary edge server
- Single application entry point
- HTTPS termination
- Reverse proxy
- Load balancing
- Forwarding requests to Backend A and Backend B

### Mac 3 — Backend B

**Team Member:** Vidhi  
**IP Address:** `10.7.7.85`

**Services:**

- Backend B → Port 3002

**Responsibilities:**

- Backend B hosting
- Processing requests forwarded by Nginx
- Providing a second backend for load balancing and failure testing

## Request Flow

```text
                         Client
                            |
                            | DNS Query
                            v
                    Mac 1 — dnsmasq
                       10.7.9.62
                            |
                            | app.teambyte.test
                            |        ↓
                            |    10.7.31.1
                            v
                    Mac 2 — Nginx
                    10.7.31.1:8443
                            |
                        HTTPS / TLS
                            |
                      Load Balancing
                         /       \
                        /         \
                       v           v
                Backend A       Backend B
             10.7.9.62:3001  10.7.7.85:3002
```

## Backend A

Backend A runs on Mac 1 using Node.js and Express.

| Parameter | Value |
|---|---|
| Host | `10.7.9.62` |
| Port | `3001` |
| Backend | `A` |

**Endpoints:**

```text
GET /
GET /api/status
```

**Run Backend A:**

```bash
cd backend-a
npm install
npm start
```

**Test Backend A:**

```bash
curl -i http://10.7.9.62:3001/api/status
```

Expected response body:

```json
{
  "backend": "A",
  "status": "ok"
}
```

The backend identifies itself using the response header:

```text
X-Backend: A
```

## Backend B

Backend B runs on Mac 3 using Node.js and Express.

| Parameter | Value |
|---|---|
| Host | `10.7.7.85` |
| Port | `3002` |
| Backend | `B` |

**Endpoints:**

```text
GET /
GET /api/status
```

**Run Backend B:**

```bash
cd backend-b
npm install
npm start
```

**Test Backend B:**

```bash
curl -i http://10.7.7.85:3002/api/status
```

Expected response body:

```json
{
  "backend": "B",
  "status": "ok"
}
```

The backend identifies itself using:

```text
X-Backend: B
```

## Private DNS

Mac 1 runs `dnsmasq` as the primary private DNS server.

The following domains are configured:

```text
app.teambyte.test → 10.7.31.1
api.teambyte.test → 10.7.31.1
```

Both domains resolve to Mac 2, where Nginx is running.

**Relevant dnsmasq configuration:**

```text
address=/app.teambyte.test/10.7.31.1
address=/api.teambyte.test/10.7.31.1
```

**Test DNS resolution:**

```bash
dig @10.7.9.62 app.teambyte.test
dig @10.7.9.62 api.teambyte.test
```

The DNS response should resolve the application domain to:

```text
10.7.31.1
```

## Nginx Reverse Proxy

Mac 2 acts as the primary edge server.

Nginx provides:

- Single application entry point
- Reverse proxy
- HTTPS termination
- Load balancing
- Backend request forwarding

**Backend servers:**

```text
Backend A → 10.7.9.62:3001
Backend B → 10.7.7.85:3002
```

**Nginx ports:**

```text
HTTP  → 8080
HTTPS → 8443
```

**Upstream configuration:**

```nginx
upstream backends {
    server 10.7.9.62:3001;
    server 10.7.7.85:3002;
}
```

## HTTPS / TLS

The project uses HTTPS with TLS 1.3.

The HTTPS service runs on port:

```text
8443
```

The certificate includes the project domains:

```text
app.teambyte.test
api.teambyte.test
```

**HTTPS test:**

```bash
curl -k -i https://app.teambyte.test:8443/api/status
```

The response identifies the backend using:

```text
X-Backend: A
```

or:

```text
X-Backend: B
```

The TLS handshake was inspected using OpenSSL and Wireshark.

## Load Balancing

Nginx distributes incoming requests between Backend A and Backend B using round-robin load balancing.

**Example:**

```text
Request 1 → Backend A
Request 2 → Backend B
Request 3 → Backend A
Request 4 → Backend B
Request 5 → Backend A
Request 6 → Backend B
```

**Test load balancing:**

```bash
for i in {1..10}; do
    curl -k -s -D - https://app.teambyte.test:8443/api/status \
    -o /dev/null | grep X-Backend
done
```

## HTTP Caching

The `/api/status` endpoint returns HTTP caching headers.

Example:

```text
Cache-Control: max-age=60
ETag: <generated-etag>
```

The ETag can be used for conditional requests.

A matching conditional request can return:

```text
HTTP/1.1 304 Not Modified
```

This demonstrates HTTP conditional caching.

## Packet Analysis

Wireshark was used to inspect:

- DNS query and response
- TCP three-way handshake
- TLS handshake
- Encrypted application data
- HTTP response headers
- Backend responses

## Backend Failure Demonstration

Backend failure handling was tested by stopping one backend while keeping the other available.

The Nginx edge continued serving requests through the available backend.

Phase 2 further extends this implementation with explicit passive failure detection and recovery testing.

---

# PHASE 2 — NETWORK REDUNDANCY, SECURITY AND HIGH AVAILABILITY

Phase 2 introduces six extensions to improve the reliability, security, availability, and troubleshooting capabilities of the original network platform.

## Extension A — Backup DNS Resolver

### Objective

To introduce DNS redundancy by configuring a secondary DNS resolver on Mac 3.

If the primary DNS resolver becomes unavailable, clients configured to use the backup resolver can still resolve the project domains.

### Architecture

```text
                  DNS Clients
                       |
              +--------+--------+
              |                 |
              v                 v
        Primary DNS        Backup DNS
          Mac 1               Mac 3
        10.7.9.62           10.7.7.85
              |                 |
              +--------+--------+
                       |
                 DNS Resolution
                       |
                 10.7.31.1
                  Nginx Edge
```

### Backup DNS Configuration

Mac 3 runs a separate dnsmasq instance.

Relevant configuration:

```text
address=/app.teambyte.test/10.7.31.1
address=/api.teambyte.test/10.7.31.1

listen-address=127.0.0.1,10.7.7.85
bind-interfaces
no-resolv

server=8.8.8.8
server=10.5.7.1
```

### Test Backup DNS

Run:

```bash
dig @10.7.7.85 app.teambyte.test
dig @10.7.7.85 api.teambyte.test
dig @10.7.7.85 google.com
```

Expected project-domain resolution:

```text
app.teambyte.test → 10.7.31.1
api.teambyte.test → 10.7.31.1
```

### Verification

- [x] Secondary dnsmasq configured on Mac 3
- [x] Backup application DNS record configured
- [x] Backup API DNS record configured
- [x] Direct DNS queries verified
- [x] External domain resolution verified

**Result:** A secondary DNS resolver was successfully configured and tested.

Note: Backup DNS is available for clients configured to use it. Automatic DNS failover between resolvers was not independently demonstrated.

---

## Extension B — DNS TTL and Controlled Record Updates

### Objective

To control how long DNS responses are cached and demonstrate DNS record changes.

TTL (Time To Live) determines how long a DNS response can remain cached before it should be queried again.

### DNS TTL Configuration

The primary dnsmasq configuration on Mac 1 includes:

```text
local-ttl=30
```

This sets the local DNS response TTL to 30 seconds.

### Test TTL

Run:

```bash
dig @10.7.9.62 app.teambyte.test
```

Expected DNS response:

```text
app.teambyte.test.  30  IN  A  10.7.31.1
```

### Controlled DNS Record Update

The DNS record was temporarily changed to point to the standby edge:

```text
app.teambyte.test → 10.7.7.85
```

The updated record was verified using:

```bash
dig @10.7.9.62 app.teambyte.test
```

After verification, the original record was restored:

```text
app.teambyte.test → 10.7.31.1
```

### Verification

- [x] DNS TTL configured to 30 seconds
- [x] TTL value verified using dig
- [x] Temporary DNS record update performed
- [x] Updated IP address verified
- [x] Original DNS record restored

**Result:** DNS TTL configuration and controlled record updates were successfully demonstrated.

Note: Retention of an old cached DNS answer until TTL expiry was not conclusively demonstrated.

---

## Extension C — Service Isolation Using PF Firewall

### Objective

To restrict direct access to backend services and allow only designated machines to communicate with them.

The project uses macOS PF (Packet Filter) firewall rules.

### Firewall Architecture

```text
                    Private LAN
                         |
             +-----------+-----------+
             |                       |
             v                       v
        Nginx Edge               Other Machines
        10.7.31.1
             |
       Allowed Access
             |
       +-----+-----+
       |           |
       v           v
   Backend A   Backend B
   10.7.9.62   10.7.7.85
    Port 3001   Port 3002
```

### Backend A Firewall Rules

Configured on Mac 1:

```text
# TeamByte - Backend A isolation

pass in quick on en0 inet proto tcp from 10.7.31.1 to 10.7.9.62 port 3001 flags S/SA keep state

pass in quick on en0 inet proto tcp from 10.7.7.85 to 10.7.9.62 port 3001 flags S/SA keep state

block in quick on en0 inet proto tcp from any to 10.7.9.62 port 3001
```

These rules allow connections from the primary edge and standby edge, while blocking other incoming connections to Backend A.

### Backend B Firewall Rules

Configured on Mac 3:

```text
# TeamByte - Backend B isolation

pass in quick on en0 inet proto tcp from 10.7.31.1 to 10.7.7.85 port 3002 flags S/SA keep state

block in quick on en0 inet proto tcp from any to 10.7.7.85 port 3002
```

These rules allow the primary edge to access Backend B and block other incoming connections.

### Firewall Testing

Check PF status:

```bash
sudo pfctl -s info
```

Display the TeamByte anchor rules:

```bash
sudo pfctl -a com.apple/teambyte -sr
```

Test an allowed connection:

```bash
nc -vz 10.7.9.62 3001
```

Test a blocked connection to Backend B from Mac 1:

```bash
nc -vz 10.7.7.85 3002
```

The unauthorized connection should fail.

### Verification

- [x] PF enabled on Mac 1
- [x] PF enabled on Mac 3
- [x] Backend A access restricted
- [x] Backend B access restricted
- [x] Primary edge access verified
- [x] Standby edge access to Backend A verified
- [x] Unauthorized connection blocked

**Result:** Firewall-based backend service isolation was successfully implemented and tested.

---

## Extension D — High Availability and Backend Failover

### Objective

To maintain application availability when one backend server becomes unavailable.

Nginx uses passive failure detection to avoid sending requests to an unavailable backend.

### Nginx Upstream Configuration

The upstream configuration on Mac 2 is:

```nginx
upstream backends {
    server 10.7.9.62:3001 max_fails=1 fail_timeout=5s;
    server 10.7.7.85:3002 max_fails=1 fail_timeout=5s;
}
```

The configuration uses:

- `max_fails=1` — one unsuccessful attempt within the failure window marks the server unavailable.
- `fail_timeout=5s` — controls the failure window and temporary unavailability period.

### Normal Operation

Both backends are available.

```text
                  Client
                     |
                     v
                 Nginx Edge
                     |
              Round Robin Routing
                 /         \
                v           v
           Backend A    Backend B
            Available    Available
```

Requests are distributed between both backends.

### Backend A Failure Simulation

Backend A was stopped to simulate a server failure.

```text
                  Client
                     |
                     v
                 Nginx Edge
                     |
                 Failover
                     |
                     v
                 Backend B
                  Available
```

The edge server continued to serve requests through Backend B.

### Test Failover

Stop Backend A on Mac 1:

```bash
kill <Backend_A_PID>
```

Verify that Backend A is unavailable:

```bash
nc -vz 10.7.9.62 3001
```

Send requests through Nginx on Mac 2:

```bash
for i in {1..6}; do
    curl -k -s https://app.teambyte.test:8443/api/status
    echo
done
```

Expected response:

```json
{
  "backend": "B",
  "status": "ok"
}
```

### Backend Recovery

Restart Backend A using its original startup command.

Verify:

```bash
curl http://10.7.9.62:3001/api/status
```

After recovery, both backends can serve requests again.

### Verification

- [x] Round-robin load balancing verified
- [x] Backend A stopped for failure simulation
- [x] Backend A unavailability verified
- [x] Backend B continued serving requests
- [x] Backend A restarted
- [x] Backend A recovery verified

**Result:** Backend failover and recovery were successfully demonstrated using Nginx passive failure detection.

Note: This implementation uses passive failure detection, not active health checks.

---

## Extension E — Controlled Edge Migration

### Objective

To configure Mac 3 as a standby edge server and demonstrate controlled migration of the application entry point from the primary edge to the standby edge.

### Edge Architecture

```text
                 Primary Edge
                   Mac 2
                 10.7.31.1
                      |
                 Normal Access
                      |
                  DNS Record
                      |
              app.teambyte.test
                      |
             Controlled Migration
                      |
                      v
                 Standby Edge
                   Mac 3
                 10.7.7.85
```

### Standby Nginx Configuration

Nginx was installed and configured on Mac 3.

The standby edge uses:

```text
HTTP  → 8080
HTTPS → 8443
```

Its upstream configuration includes both backend servers:

```nginx
upstream backends {
    server 10.7.9.62:3001 max_fails=1 fail_timeout=5s;
    server 10.7.7.85:3002 max_fails=1 fail_timeout=5s;
}
```

### TLS Certificate Migration

The TLS certificate and private key were transferred from Mac 2 to Mac 3.

Certificate location:

```text
/opt/homebrew/etc/nginx/certs/teambyte.crt
```

Private key location:

```text
/opt/homebrew/etc/nginx/certs/teambyte.key
```

Permissions:

```text
Certificate → 644
Private Key → 600
```

### Verify Standby Edge

Test HTTPS directly:

```bash
curl -k https://10.7.7.85:8443/api/status
```

Test HTTP:

```bash
curl http://10.7.7.85:8080/api/status
```

### Controlled DNS Migration

During migration, the primary DNS records on Mac 1 were temporarily changed:

```text
app.teambyte.test → 10.7.7.85
api.teambyte.test → 10.7.7.85
```

The configuration was tested and dnsmasq restarted.

The application was then accessed through the migrated domain.

After testing, the original DNS records were restored:

```text
app.teambyte.test → 10.7.31.1
api.teambyte.test → 10.7.31.1
```

### Verification

- [x] Nginx installed on Mac 3
- [x] Standby edge configured
- [x] TLS certificate transferred
- [x] Nginx configuration validated
- [x] Standby HTTP access verified
- [x] Standby HTTPS access verified
- [x] DNS records changed for migration
- [x] Application access through migrated domain verified
- [x] Original DNS records restored

**Result:** Controlled edge migration from Mac 2 to Mac 3 was successfully demonstrated.

Note: Edge migration is manually controlled through DNS updates. Automatic edge election or automatic DNS failover was not implemented.

---

## Extension F — Network Troubleshooting

### Objective

To establish a healthy network baseline and verify the different layers involved in application communication.

The troubleshooting process checks:

- DNS resolution
- TCP connectivity
- TLS handshake
- Application response

### 1. DNS Verification

Test:

```bash
dig @10.7.9.62 app.teambyte.test
```

Expected result:

```text
app.teambyte.test → 10.7.31.1
```

### 2. TCP Connectivity Verification

Test:

```bash
nc -vz app.teambyte.test 8443
```

Expected result:

```text
Connection to app.teambyte.test port 8443 succeeded
```

### 3. TLS Verification

Test:

```bash
openssl s_client \
-connect app.teambyte.test:8443 \
-servername app.teambyte.test
```

This verifies the TLS connection and displays handshake and certificate information.

The test observed a TLS 1.3 handshake. The certificate is self-signed, so OpenSSL reports a certificate verification warning unless the certificate is trusted by the client.

### 4. Application Verification

Test:

```bash
curl -k -v https://app.teambyte.test:8443/api/status
```

Expected response:

```text
HTTP/1.1 200 OK
X-Backend: A
```

or:

```text
HTTP/1.1 200 OK
X-Backend: B
```

Expected response body:

```json
{
  "backend": "B",
  "status": "ok"
}
```

The backend identity may vary depending on load balancing and backend availability.

### Healthy Baseline Verification

- [x] DNS resolution verified
- [x] TCP connectivity verified
- [x] TLS handshake verified
- [x] Application HTTP 200 response verified

**Result:** A healthy network baseline was successfully established across DNS, TCP, TLS, and application layers.

Note: The actual faculty-injected fault scenario is pending the evaluation exercise.

---

# HTTP CACHING AND PACKET CAPTURE EVIDENCE

## HTTP Caching Verification

The HTTP endpoint was tested through the edge server.

Example:

```bash
curl -i http://10.7.31.1:8080/api/status
```

The response included:

```text
HTTP/1.1 200 OK
Cache-Control: max-age=60
ETag: <generated-etag>
X-Backend: A
```

The caching-related headers were verified. Actual browser cache reuse was not independently demonstrated.

## Wireshark Packet Analysis

Wireshark was used to capture and inspect network traffic.

| Capture File | Purpose |
|---|---|
| `Phase2_DNS_Capture.pcapng` | DNS query and response |
| `Phase2_TCP_Capture.pcapng` | TCP connection establishment |
| `Phase2_TLS_Capture.pcapng` | TLS handshake |
| `Phase2_HTTP_Caching_Capture.pcapng` | HTTP response and caching headers |

### DNS Capture

Display filter:

```text
dns.qry.name == "app.teambyte.test"
```

Observed:

```text
DNS Query: app.teambyte.test
DNS Response: 10.7.31.1
TTL: 30
```

### TCP Capture

Display filter:

```text
tcp.port == 8443
```

For connection establishment:

```text
tcp.port == 8443 && (tcp.flags.syn == 1 || tcp.flags.ack == 1)
```

Observed TCP connection establishment between client and Nginx.

### TLS Capture

Display filter:

```text
tls.handshake
```

Observed TLS handshake messages including:

```text
ClientHello
ServerHello
```

After the handshake, application data is encrypted.

### HTTP Caching Capture

The HTTP response was inspected for:

```text
Cache-Control
ETag
X-Backend
```

These headers demonstrate the HTTP response and caching policy configured for the application.

---

# Repository Structure

```text
TeamByte/
│
├── README.md
│
├── Report/
│   └── TeamByte_Phase2_Final_Report.pdf
│
├── backend-a/
│   ├── server.js
│   ├── package.json
│   └── package-lock.json
│
├── backend-b/
│   ├── server.js
│   ├── package.json
│   └── package-lock.json
│
├── nginx/
│   └── nginx.conf
│
├── dns/
│   ├── dnsmasq.conf
│   └── backup-dnsmasq.conf
│
├── tls/
│   └── README.md
│
└── Evidence/
    │
    ├── Firewall/
    │   ├── Backend_A_Firewall_Rules.png
    │   ├── Backend_A_Allowed.png
    │   └── Backend_B_Blocked.png
    │
    ├── Failover/
    │   ├── Round_Robin_A_B.png
    │   ├── Backend_A_Down_B_Serving.png
    │   └── Backend_A_Restored.png
    │
    ├── Edge_Migration/
    │   ├── Standby_Edge_HTTPS.png
    │   ├── Standby_Edge_HTTP.png
    │   ├── DNS_Pointing_To_Standby.png
    │   ├── Application_Access_Through_Migrated_Domain.png
    │   └── Original_Edge_Restored.png
    │
    ├── Troubleshooting/
    │   ├── DNS_Healthy.png
    │   ├── TCP_Healthy.png
    │   ├── TLS_Healthy.png
    │   └── Application_Healthy.png
    │
    ├── DNS/
    │
    └── Wireshark/
        ├── Phase2_DNS_Capture.pcapng
        ├── Phase2_TCP_Capture.pcapng
        ├── Phase2_TLS_Capture.pcapng
        └── Phase2_HTTP_Caching_Capture.pcapng
```

*The repository structure represents the recommended organization of the project report and collected evidence. Include only files that actually exist in the repository.*

---

# Technologies Used

- macOS
- Node.js
- Express.js
- Nginx
- dnsmasq
- PF (Packet Filter)
- HTTPS / TLS 1.3
- Wireshark
- curl
- dig
- nc
- OpenSSL
- TCP/IP
- Private IPv4 Networking

---

# Phase 2 Verification Checklist

| Extension | Implementation | Status |
|---|---|---|
| A. Backup DNS | Secondary dnsmasq resolver | Completed |
| B. DNS TTL | TTL configuration and record update | Completed |
| C. Service Isolation | PF firewall rules | Completed |
| D. Backend Failover | Nginx passive failure detection | Completed |
| E. Edge Migration | Standby Nginx and controlled DNS migration | Completed |
| F. Troubleshooting | Healthy baseline across DNS, TCP, TLS and application | Baseline completed; faculty fault pending |

---

# Implementation Limitations

The following limitations are documented to distinguish the demonstrated functionality from features that were not independently verified:

1. Backend failover uses Nginx passive failure detection rather than active health checks.
2. Edge migration is manually controlled through DNS record updates.
3. Automatic DNS failover between primary and backup resolvers was not independently demonstrated.
4. Old DNS cache retention until TTL expiry was not conclusively demonstrated.
5. Actual browser cache reuse was not independently tested.
6. The faculty-injected troubleshooting fault scenario remains pending the evaluation exercise.
7. The TLS certificate is self-signed; clients need to trust it to avoid certificate verification warnings.

---

# Conclusion

The Private Network Service Platform demonstrates the practical implementation of important Computer Networks concepts using three interconnected macOS machines.

Phase 1 established the foundation through private DNS, TCP/IP communication, HTTP, HTTPS/TLS, reverse proxying, load balancing, HTTP caching, packet analysis, and backend failure handling.

Phase 2 extended the system with DNS redundancy, TTL management, firewall-based service isolation, backend failover, controlled edge migration, and structured network troubleshooting.

Together, both phases demonstrate how networking services can be configured, secured, monitored, and made more resilient in a private LAN environment.

**Project:** Private Network Service Platform  
**Course:** Computer Networks  
**Team:** Vani Rudra, Apoorva Choudhary, Vidhi  
**Environment:** Three macOS machines connected over a private LAN