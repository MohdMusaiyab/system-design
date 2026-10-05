# Load Balancer Redundancy & High Availability

A load balancer sits between clients and backend servers. If there is only one load balancer and it fails, the backend servers may still be healthy, but clients cannot reach them.

Therefore, in production systems, load balancers are usually deployed with **redundancy and failover**.

```mermaid
flowchart LR
    C[Clients] --> LB[Load Balancer]
    LB --> A[Backend A]
    LB --> B[Backend B]
    LB --> D[Backend C]

    X[LB Failure] -.-> LB
```

---

## Single Point of Failure

A **Single Point of Failure (SPOF)** is a component whose failure can make the entire system unavailable.

A single load balancer can become an SPOF:

```mermaid
flowchart LR
    C[Clients] --> LB[Single Load Balancer]

    LB --> A[Server A]
    LB --> B[Server B]

    LB -. Failure .-> X[Backend Unreachable]
```

Even though multiple backend servers exist, the load balancer is still a single failure point.

**Goal:**

> Removing one load balancer should not make the service unavailable.

---

## LB Redundancy

**LB redundancy** means deploying multiple load balancers so that one can continue handling traffic if another fails.

```mermaid
flowchart LR
    C[Clients] --> LB1[Load Balancer 1]
    C --> LB2[Load Balancer 2]

    LB1 --> A[Backend A]
    LB1 --> B[Backend B]

    LB2 --> A
    LB2 --> B
```

The two common architectures are:

- **Active-Active**
- **Active-Passive**

Redundancy removes the load balancer as a single point of failure.

---

## Active-Active Load Balancers

In **Active-Active**, multiple load balancers actively serve traffic at the same time.

```mermaid
flowchart LR
    C[Clients] --> LB1[LB 1]
    C --> LB2[LB 2]

    LB1 --> A[Backend]
    LB1 --> B[Backend]

    LB2 --> A
    LB2 --> B
```

Traffic can be distributed between both load balancers.

### Advantages

- Both LBs are utilized.
- Better capacity.
- No idle standby LB.
- Failure of one LB can be handled by the other.

### Challenge

The system needs a way to distribute traffic between the load balancers, such as:

- DNS
- Anycast
- Another network-level mechanism
- Cloud-managed infrastructure

---

## Active-Passive Load Balancers

In **Active-Passive**, one load balancer handles traffic while another waits as a standby.

```mermaid
flowchart LR
    C[Clients] --> VIP[Virtual / Floating IP]

    VIP --> LB1[Active LB]
    VIP -. Failover .-> LB2[Passive LB]

    LB1 --> A[Backend]
    LB1 --> B[Backend]

    LB2 --> A
    LB2 --> B
```

If the active LB fails, the passive LB takes over.

### Advantages

- Simpler traffic model.
- Easier state management in some systems.
- Clear failover path.

### Disadvantage

The standby LB normally does not handle production traffic, so some capacity remains unused.

---

## LB Failover

**Failover** is the process of moving traffic from a failed load balancer to a healthy one.

Typical flow:

```mermaid
sequenceDiagram
    participant C as Clients
    participant LB1 as Active LB
    participant LB2 as Standby LB

    C->>LB1: Request
    LB1-->>C: Response

    Note over LB1: LB1 fails

    LB2->>LB2: Detect failure
    LB2->>LB2: Take ownership
    C->>LB2: Request
    LB2-->>C: Response
```

Failover depends on:

1. Detecting that the active LB failed.
2. Selecting a healthy LB.
3. Moving traffic to it.
4. Ensuring the new LB is ready to serve traffic.

**Failover time** is important because users may experience errors during the transition.

---

## Virtual/Floating IP

A **Virtual IP (VIP)** is an IP address that represents the load-balancing service rather than a specific physical/virtual machine.

For example:

```text
VIP: 10.0.0.100

        ↓

   Active LB
```

If the active LB fails:

```text
VIP: 10.0.0.100

        ↓

   New Active LB
```

The important point is that **clients continue connecting to the same IP**.

The IP ownership moves between the load balancers.

This is especially useful in active-passive architectures.

---

## Health Checking Load Balancers

Redundant LBs need a mechanism to determine whether another LB is healthy.

This is different from **backend health checks**.

```text
LB Health Check
      │
      ▼
Is LB1 alive and able to serve traffic?
```

The system may check:

- Process/service availability
- Network connectivity
- Control endpoint
- Ability to forward traffic
- Overall LB health

If the active LB becomes unhealthy, failover can be triggered.

```mermaid
flowchart LR
    H[Health Checker] --> LB1[Active LB]
    H --> LB2[Standby LB]

    LB1 -->|Healthy| T[Continue Traffic]
    LB1 -->|Failed| F[Trigger Failover]
    F --> LB2
```

Health checking should avoid declaring a healthy LB failed because of a temporary network issue.

---

## State Synchronization

Some load balancers maintain state such as:

- Connection information
- Session information
- NAT mappings
- Configuration
- Routing state

If two LBs need this information, they may synchronize state.

```mermaid
flowchart LR
    LB1[LB 1] <--> S[State Synchronization] <--> LB2[LB 2]
```

State synchronization can make failover smoother because the new LB already knows relevant state.

However, not every system needs to synchronize all state.

A **stateless load balancer** is easier to fail over because there is less state to transfer.

> The less critical state the LB owns, the simpler redundancy usually becomes.

---

## Multi-AZ Load Balancing

A highly available system should avoid placing all load balancers in the same **Availability Zone (AZ)**.

For example:

```mermaid
flowchart TB
    C[Clients]

    C --> LB1[LB - AZ 1]
    C --> LB2[LB - AZ 2]

    LB1 --> A[Backend - AZ 1]
    LB1 --> B[Backend - AZ 2]

    LB2 --> A
    LB2 --> B
```

If an entire AZ becomes unavailable, the load balancer in another AZ can continue serving traffic.

This protects against **zone-level failures**, not just individual machine failures.

---

## LB High Availability

**High Availability (HA)** means designing the load-balancing layer so that failure of an individual component does not cause service-wide downtime.

A typical HA architecture may look like:

```mermaid
flowchart TB
    C[Clients]

    C --> G[Traffic Distribution / VIP]

    G --> LB1[LB - AZ 1]
    G --> LB2[LB - AZ 2]

    LB1 --> A[Backend - AZ 1]
    LB1 --> B[Backend - AZ 2]

    LB2 --> A
    LB2 --> B
```

High availability usually combines:

- Multiple load balancers
- Active-active or active-passive architecture
- Health checking
- Failover
- Multi-AZ deployment
- Appropriate state handling

### Important Trade-off

More redundancy improves availability but also increases:

- Infrastructure cost
- Configuration complexity
- Operational complexity
- State-management requirements

The goal is not simply to add more load balancers, but to **remove meaningful failure points without creating unnecessary complexity**.

## Key Takeaways

- A single LB can become a **Single Point of Failure**.
- **Redundant LBs** prevent one LB failure from taking down the service.
- **Active-Active** → multiple LBs serve traffic simultaneously.
- **Active-Passive** → one serves traffic while another waits for failover.
- **Failover** moves traffic from a failed LB to a healthy one.
- A **Virtual/Floating IP** can allow the service IP to move between LBs.
- **Health checks** detect LB failures.
- **State synchronization** can make failover easier when LBs maintain important state.
- **Multi-AZ deployment** protects against entire availability-zone failures.
- HA is a combination of **redundancy + failure detection + failover + appropriate architecture**.

---


