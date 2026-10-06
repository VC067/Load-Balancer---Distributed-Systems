# Distributed Load Balancer with Consistent Hashing & Docker Replicas

**Course:** B.Tech — Cloud Computing & Distributed Systems  
**Project:** Distributed Load Balancer  
**Technologies:** Python 3.10, Flask, Docker, Consistent Hashing, APScheduler  
**Dashboard URL:** `http://localhost:6000/`  

---

## 1. What is the Problem?

When web services grow, handling high traffic on a single server causes three critical problems:

1. **Single Point of Failure (SPOF) & Server Overload:**  
   If all clients connect to only one server, that server quickly exhausts its CPU, memory, and network capacity. If that single server crashes, the entire service goes completely offline.
   
2. **The Flaw of Traditional Modulo Load Balancing (`Hash % N`):**  
   A naive load balancer distributes requests across $N$ servers using simple division remainder:
   $$\text{Server} = \text{Hash}(\text{Request ID}) \pmod N$$
   When traffic increases and you add a new server ($N=3 \to 4$), the divisor changes for every key. **Almost 100% of all requests are remapped to completely different servers.** This wipes out server caches, forces heavy database queries, and can crash the backend database cluster.

3. **Silent Server Crashes (No Fault Tolerance):**  
   If one server replica crashes silently, a basic system does not notice and keeps forwarding client requests to the dead server, returning errors to users.

---

## 2. How Are We Solving It?

We built a **Distributed Load Balancer** that coordinates multiple server replicas using **Consistent Hashing** and **Docker containerization**:

| The Problem | How We Solve It | Technical Mechanism |
| :--- | :--- | :--- |
| **Single Point of Failure** | **Server Replicas (Docker)** | We run multiple identical server replicas (`Server-1`, `Server-2`, `Server-3`) in isolated Docker containers. If one replica fails, the others continue serving traffic. |
| **Modulo Remapping Flaw** | **Consistent Hashing** | We map servers and requests onto a circular 512-slot hash ring with virtual nodes ($K=20$). Adding or removing a server only remaps a small fraction ($\sim 1/N$) of requests, protecting cache affinity. |
| **Silent Server Crashes** | **Heartbeat Fault Detection** | The load balancer probes `/heartbeat` on every server replica every 30 seconds. Dead containers are automatically detected, purged from the hash ring, and deleted via `docker rm -f`. |
| **Traffic Spikes** | **Dynamic Horizontal Scaling** | REST endpoints (`/add`, `/rm`) allow spinning up or terminating Docker containers dynamically in seconds. |

---

## 3. System Architecture

```
        Client Requests (Browser / cURL)
                       │
                       ▼
         ┌───────────────────────────┐
         │   Load Balancer (Port 6000)│
         │   - Consistent Hash Ring  │
         │   - Heartbeat Checker (30s)│
         └─────────────┬─────────────┘
                       │ Docker Bridge Network (load_balancer_network)
           ┌───────────┼───────────┐
           ▼           ▼           ▼
     ┌───────────┐┌───────────┐┌───────────┐
     │ Server-1  ││ Server-2  ││ Server-3  │
     │ (Docker)  ││ (Docker)  ││ (Docker)  │
     └───────────┘└───────────┘└───────────┘
```

1. **Client** sends an HTTP request to the **Load Balancer** on port `6000`.
2. The **Load Balancer** hashes the request onto the **Consistent Hash Ring** and selects the corresponding **Server Replica**.
3. The selected **Server Replica** runs in its own isolated Docker container and returns the response.
4. An **APScheduler Background Job** checks server health every 30 seconds via `/heartbeat`.

---

## 4. Consistent Hashing Explained Simply

Instead of using basic modulo division (`Request % N`), Consistent Hashing uses a **circular ring of 512 slots (indices 0 to 511)**:

1. **Placing Server Replicas on the Ring:**  
   Each server replica is placed at multiple virtual positions around the ring (**Virtual Nodes**, $K = 20$). This spreads traffic evenly and prevents one server from getting too many requests.

2. **Routing Client Requests:**  
   When a request arrives, its Request ID is hashed to a slot on the ring:
   $$H(\text{Request ID}) = (\text{ID}^2 + 2\cdot\text{ID} + 17) \pmod{512}$$
   The load balancer moves **clockwise** from that slot until it hits the first server replica. That server handles the request.

3. **Why Consistent Hashing is Superior:**  
   When a new server is added:
   - Only a small slice of the ring is assigned to the new server.
   - All other servers keep their existing requests.
   - **Cache disruption is minimized to $\sim 1/N$ instead of 100%.**

---

## 5. Technologies Used & Why

* **Python 3.10:** Implements the core load balancer routing and consistent hashing algorithms.
* **Flask:** Provides the lightweight REST API framework for request routing, management, and GUI endpoints.
* **Docker:** Runs each server replica in an isolated Linux container with its own process space and environment variable (`SERVER_ID`).
* **Docker Bridge Networking:** Connects the load balancer and server containers on `load_balancer_network` with internal DNS hostname resolution.
* **APScheduler:** Runs non-blocking background heartbeat checks every 30 seconds.
* **Asyncio & HTTPX:** Concurrently probes all server replicas in parallel without blocking incoming client requests.
* **Matplotlib:** Generates distribution bar charts on-the-fly and streams them via `/graph`.

---

## 6. REST API Reference

| Endpoint | Method | Purpose |
| :--- | :---: | :--- |
| `/home` | GET | Client endpoint. Routes request to a server replica using consistent hashing. |
| `/rep` | GET | Returns the list and count of all active, healthy server replicas. |
| `/add` | POST | Horizontally scales up by spawning new Docker server containers. |
| `/rm` | POST / DELETE | Horizontally scales down by stopping and removing Docker server containers. |
| `/checkpoint` | GET | Returns request counts handled by each server replica. |
| `/graph` | GET | Returns a Matplotlib-generated bar chart image of traffic distribution. |
| `/ring` | GET | Returns virtual node slot coordinates on the 512-slot ring. |
| `/` | GET | Opens the interactive presentation GUI dashboard. |

---

## 7. Fault Tolerance & Self-Healing

The load balancer implements automatic failure recovery:

1. **Failure Simulation:** A container crashes or is stopped manually (e.g. `Server-2`).
2. **Heartbeat Detection:** Every 30 seconds, the load balancer sends an HTTP GET request to `http://<server>:5000/heartbeat`.
3. **Dead Container Removal:** `Server-2` fails the heartbeat check.
4. **Automated Recovery Actions:**
   - The load balancer marks `Server-2` as unhealthy.
   - It removes `Server-2`'s virtual nodes from the consistent hash ring.
   - It issues `docker rm -f Server-2` to delete the dead container.
   - Subsequent client requests are automatically handled by the remaining healthy servers (`Server-1`, `Server-3`).
5. **Outcome:** Zero client downtime during server crashes.

---

## 8. How to Run the Project

```bash
# 1. Create the Docker bridge network
docker network create load_balancer_network

# 2. Build the Server replica Docker image
docker build -t server -f Dockerfile.server .

# 3. Build the Load Balancer Docker image
docker build -t server_image .

# 4. Run the Load Balancer container
docker run -d \
  --name load_balancer \
  -p 6000:5000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --network load_balancer_network \
  server_image

# 5. Open the Presentation Dashboard
open http://localhost:6000/
```

---

## 9. Viva / Presentation Q&A Cheat Sheet

**Q1: What is a Load Balancer?**  
*A: A system that distributes incoming client requests across multiple backend server replicas so no single server becomes a bottleneck.*

**Q2: What is the main problem with standard Modulo Load Balancing (`Hash % N`)?**  
*A: When you add or remove a server ($N$ changes), almost 100% of all request keys are remapped, destroying caches and overloading databases. Consistent Hashing solves this by only moving $\sim 1/N$ of keys.*

**Q3: What are Virtual Nodes in Consistent Hashing?**  
*A: Instead of placing each physical server at only 1 spot on the ring, we place it at $K=20$ virtual positions. This ensures traffic is distributed evenly and prevents clustering.*

**Q4: Why use Docker containers in this project?**  
*A: Docker allows us to run multiple isolated server replicas on a single machine, each with its own network identity, port, and environment variables.*

**Q5: How does Heartbeat Fault Tolerance work?**  
*A: The load balancer polls `/heartbeat` on every server replica every 30 seconds. If a replica stops responding, it is automatically removed from the consistent hash ring and its container is deleted.*

---
