# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEMYSTIFY FIRST, SERVICES SECOND                               │
│  ────────────────────────────────                               │
│  Students think "the cloud" is alien technology.                │
│  We shatter that illusion in the first 10 minutes.             │
│                                                                 │
│  MAP TO WHAT THEY BUILT                                         │
│  ──────────────────────                                         │
│  Every cloud service maps to something in their                 │
│  docker-compose.yml. No service is introduced in a vacuum.      │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Cloud infrastructure = Housing.                                │
│  From living with parents → renting → hotel stays.              │
│  Every decision maps to a real-world housing choice.            │
│                                                                 │
│  CONCEPTS OVER CONSOLES                                         │
│  ─────────────────────                                          │
│  We don't click through AWS dashboards. We build                │
│  transferable mental models that apply to ANY provider.         │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Docker (L1) → containers run on cloud compute                  │
│  pydantic-settings (L2) → same config, different env            │
│  CI/CD (L3) → the pipeline deploys TO these services            │
│  Auth/RBAC (Week 9) → IAM is RBAC for infrastructure           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLOUD FUNDAMENTALS                           │
│                    (3-4 Hour Lecture)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 Your Docker Compose in "Production"                     │
│  ├─ 1.2 Three Problems Your Laptop Can't Solve                  │
│  ├─ 1.3 What the Cloud Actually Is                              │
│  └─ 1.4 The Housing Analogy                                     │
│                                                                 │
│  PART 2: THE MAP (30 min)                                       │
│  ├─ 2.1 The Big Three Providers                                 │
│  ├─ 2.2 The Rosetta Stone (Same Concepts, Different Names)      │
│  ├─ 2.3 docker-compose.yml → Cloud Translation                  │
│  └─ 2.4 Only Five Things Matter                                 │
│                                                                 │
│  PART 3: CORE SERVICES (60 min)                                 │
│  ├─ 3.1 Compute — Where Your Code Runs                          │
│  ├─ 3.2 Database — Managed Persistence                          │
│  ├─ 3.3 Storage — Objects and Files                              │
│  ├─ 3.4 Networking — VPC Basics                                  │
│  └─ 3.5 IAM — The Permission System                             │
│                                                                 │
│  PART 4: THE REAL DECISIONS (40 min)                            │
│  ├─ 4.1 The Responsibility Spectrum                             │
│  ├─ 4.2 Managed vs Self-Hosted (Three Case Studies)             │
│  ├─ 4.3 Infrastructure as Code (Terraform Awareness)            │
│  └─ 4.4 Reading Terraform (Literacy, Not Fluency)               │
│                                                                 │
│  PART 5: COST AWARENESS & SERVERLESS (30 min)                   │
│  ├─ 5.1 How Cloud Billing Actually Works                        │
│  ├─ 5.2 The Cost Traps That Burn Beginners                      │
│  ├─ 5.3 Serverless — Functions as a Service                     │
│  ├─ 5.4 When Serverless Fits (and When It Doesn't)              │
│  └─ 5.5 Your Deployment Decision Framework                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 Your Docker Compose in "Production"

**Start with what they already have. Open the capstone docker-compose.yml.**

```yaml
# Their capstone project — they've been running this locally for weeks
# (You will have your own version — this is representative)

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:pass@db:5432/saas
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  worker:
    build: .
    command: celery -A app.worker worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:pass@db:5432/saas
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  beat:
    build: .
    command: celery -A app.worker beat --loglevel=info
    depends_on:
      - redis

  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=pass

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

**Now ask the class:**

> "This runs beautifully on your machine. `docker compose up` — everything works. Your API handles requests, Celery processes background jobs, Postgres stores data, Redis caches results. Let's say you pass the course, get a job, and your boss says: 'Ship this to production. Real users need to access it.' What do you do?"

**Pause. Let them think. Then ask the real questions:**

> "Question one: What happens to your users when you close your laptop to go to sleep?"

> "Question two: Someone in Tokyo wants to use your API. How do they reach `localhost:8000` on your laptop in your bedroom?"

> "Question three: Your SaaS gets featured on Hacker News. 10,000 users hit your API at once. Your laptop has 16GB of RAM. What happens?"

---

## 1.2 Three Problems Your Laptop Can't Solve

```
┌─────────────────────────────────────────────────────────────────┐
│              THREE PROBLEMS, ONE SOLUTION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM 1: AVAILABILITY                                        │
│  ────────────────────────                                       │
│  Your app must run 24/7/365.                                    │
│  You close your laptop → app goes down.                         │
│  Your power goes out → app goes down.                           │
│  Your OS updates and reboots → app goes down.                   │
│                                                                 │
│  Production requirement: 99.9% uptime = 8.7 hours downtime/year│
│                                                                 │
│                                                                 │
│  PROBLEM 2: ACCESSIBILITY                                       │
│  ─────────────────────────                                      │
│  Your laptop is behind a home router with NAT.                  │
│  No public IP address. No DNS. No TLS certificate.              │
│  Users can't type "myapp.com" and reach your machine.           │
│                                                                 │
│  Production requirement: Public endpoint with HTTPS             │
│                                                                 │
│                                                                 │
│  PROBLEM 3: SCALABILITY                                         │
│  ─────────────────────────                                      │
│  Your laptop: 8 cores, 16GB RAM.                                │
│  That's the ceiling. You can't add more.                        │
│  10,000 concurrent users? Not a chance.                         │
│                                                                 │
│  Production requirement: Handle variable load                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "You need a computer that's always on, has a public address on the internet, and can grow when demand spikes. You COULD buy a server, colocate it in a data center, run cables, configure networking, hire someone to replace failed hard drives at 3 AM... OR you could rent all of that from someone who already built it."

---

## 1.3 What the Cloud Actually Is

**Destroy the mystique:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE CLOUD, DEMYSTIFIED                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│         "The Cloud" = Someone else's computers,                 │
│                        rented by the hour,                      │
│                        accessed over the internet.              │
│                                                                 │
│                        That's it.                               │
│                                                                 │
│                                                                 │
│  What AWS/GCP/Azure actually operate:                           │
│                                                                 │
│  ┌──────────────────────────────────────────────────┐           │
│  │              DATA CENTER                          │           │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐    │           │
│  │  │ Server │ │ Server │ │ Server │ │ Server │    │           │
│  │  │ Rack   │ │ Rack   │ │ Rack   │ │ Rack   │    │           │
│  │  └────────┘ └────────┘ └────────┘ └────────┘    │           │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐    │           │
│  │  │ Server │ │ Server │ │ Server │ │ Server │    │           │
│  │  │ Rack   │ │ Rack   │ │ Rack   │ │ Rack   │    │           │
│  │  └────────┘ └────────┘ └────────┘ └────────┘    │           │
│  │                                                   │           │
│  │  + Power generators + Cooling systems             │           │
│  │  + Fiber optic cables + Security guards           │           │
│  │  + 24/7 operations staff                          │           │
│  └──────────────────────────────────────────────────┘           │
│                         │                                       │
│                   × 100+ data centers worldwide                 │
│                                                                 │
│  They sliced these computers into smaller pieces               │
│  (virtualization) and rent each piece to you by the hour.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "When you run `docker compose up` locally, your operating system allocates CPU time, RAM, and disk space for each container. Cloud providers do the same thing — just on THEIR hardware instead of yours. Your Docker image? It runs on their machines. Your Postgres data? It's stored on their disks. That's the entire concept."

---

## 1.4 The Housing Analogy

**This analogy carries through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE HOUSING ANALOGY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  🏠 LIVING WITH PARENTS (Local Development)                     │
│  ──────────────────────────────────────────                     │
│  Free. Comfortable. Everything is right there.                  │
│  But you can't invite 10,000 guests over.                       │
│  And when you leave the house, nobody's home.                   │
│                                                                 │
│  = docker compose up on your laptop                             │
│                                                                 │
│                                                                 │
│  🏚️ RENTING AN EMPTY HOUSE (IaaS — EC2)                        │
│  ──────────────────────────────────────────                     │
│  You get four walls, a roof, plumbing connections.              │
│  You buy all the furniture. YOU fix the leaky faucet.           │
│  You install your own locks. Maximum control.                   │
│  Maximum responsibility.                                        │
│                                                                 │
│  = Virtual machine. You install OS, Postgres, everything.       │
│                                                                 │
│                                                                 │
│  🏢 RENTING A FURNISHED APARTMENT (Managed Services — RDS)      │
│  ──────────────────────────────────────────────────────         │
│  Furniture included. Landlord fixes the plumbing.               │
│  Landlord replaces the water heater when it breaks.             │
│  You just live in it. Less control, fewer headaches.            │
│                                                                 │
│  = Managed database. AWS handles backups, patches, failover.    │
│                                                                 │
│                                                                 │
│  🏨 HOTEL ROOM (PaaS / Serverless — Railway, Lambda)            │
│  ──────────────────────────────────────────────────             │
│  Show up. Use it. Leave. Pay per night.                         │
│  You don't even think about plumbing or furniture.              │
│  Expensive if you live there permanently.                       │
│  Perfect for short stays.                                       │
│                                                                 │
│  = Push code, it runs. Scales automatically. Pay per request.   │
│                                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy explicitly:**

```
Housing Concept              │  Cloud Concept
─────────────────────────────│──────────────────────────────
Landlord                     │  Cloud provider (AWS, GCP)
Rent                         │  Monthly cloud bill
Utility bills                │  Data transfer, storage costs
Lease agreement              │  Service Level Agreement (SLA)
Building security            │  IAM (permissions)
Gated community              │  VPC (private network)
Storage unit down the street │  S3 (object storage)
Landlord fixes plumbing      │  Managed service (auto-backups)
You fix your own plumbing    │  Self-hosted (you patch Postgres)
Square footage / rooms       │  vCPUs / RAM (instance size)
Address that guests can find │  Public IP / DNS name
```

> "Every decision you make about cloud infrastructure is a housing decision: How much control do I need? How much responsibility am I willing to take? How much am I willing to pay for convenience?"

---

# PART 2: THE MAP

## 2.1 The Big Three Providers

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE BIG THREE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────┐                                       │
│  │    AWS                │  Amazon Web Services                  │
│  │    (Amazon)           │  • Largest market share (~31%)        │
│  │                       │  • Most services (200+)               │
│  │                       │  • Most job listings mention AWS      │
│  │                       │  • We'll use AWS as primary examples  │
│  └──────────────────────┘                                       │
│                                                                 │
│  ┌──────────────────────┐                                       │
│  │    GCP                │  Google Cloud Platform                │
│  │    (Google)           │  • Strong in data/ML/Kubernetes       │
│  │                       │  • Cloud Run is excellent for         │
│  │                       │    containerized apps (like yours)    │
│  │                       │  • ~11% market share                  │
│  └──────────────────────┘                                       │
│                                                                 │
│  ┌──────────────────────┐                                       │
│  │    Azure              │  Microsoft Azure                     │
│  │    (Microsoft)        │  • Dominates enterprise/corporate     │
│  │                       │  • Strong .NET ecosystem              │
│  │                       │  • ~24% market share                  │
│  └──────────────────────┘                                       │
│                                                                 │
│  THE KEY INSIGHT:                                               │
│  All three offer the same CATEGORIES of services.               │
│  Learn the concepts on one → transfer to any.                   │
│  AWS names ≠ GCP names ≠ Azure names, but                       │
│  the underlying IDEAS are identical.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 The Rosetta Stone (Same Concepts, Different Names)

**This is the cheat sheet. Concepts transfer, names don't.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE ROSETTA STONE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT YOU NEED        AWS             GCP              AZURE    │
│  ──────────────       ────            ────             ─────    │
│                                                                 │
│  Virtual Machine      EC2             Compute Engine   VM       │
│                                                                 │
│  Run Container        ECS/Fargate     Cloud Run        Container│
│                                                         Apps    │
│                                                                 │
│  Managed Postgres     RDS             Cloud SQL        Azure DB │
│                                                        for PG   │
│                                                                 │
│  Managed Redis        ElastiCache     Memorystore      Azure    │
│                                                        Cache    │
│                                                                 │
│  Object Storage       S3              Cloud Storage    Blob     │
│                                                        Storage  │
│                                                                 │
│  Serverless Funcs     Lambda          Cloud Functions  Azure    │
│                                                        Functions│
│                                                                 │
│  Private Network      VPC             VPC              VNet     │
│                                                                 │
│  Permissions          IAM             IAM              Entra ID │
│                                                                 │
│  DNS                  Route 53        Cloud DNS        Azure DNS│
│                                                                 │
│  Load Balancer        ALB/NLB         Cloud LB         Azure LB │
│                                                                 │
│  Container Registry   ECR             Artifact Reg.    ACR      │
│                                                                 │
│  Secrets              Secrets Mgr     Secret Mgr       Key Vault│
│                                                                 │
│  Monitoring           CloudWatch      Cloud Monitoring  Monitor │
│                                                                 │
│  CDN                  CloudFront      Cloud CDN        Front    │
│                                                        Door     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Don't memorize this table. Understand that every provider has a compute service, a managed database, an object store, a permission system, and a private network. The names are just branding. If you understand ONE provider deeply, you can learn another in a week."

**For this course, we use AWS terminology as the primary reference** — it has the largest market share and appears most frequently in job postings. Every concept we discuss applies equally to GCP and Azure.

---

## 2.3 docker-compose.yml → Cloud Translation

**This is the centerpiece visual. Everything clicks here.**

```
┌─────────────────────────────────────────────────────────────────┐
│        YOUR DOCKER COMPOSE    →    AWS CLOUD                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  services:                                                      │
│                                                                 │
│    api:                       →    ECS Fargate                   │
│      build: .                      Your container, running on   │
│      ports: ["8000:8000"]          AWS hardware. Public endpoint.│
│                                    Auto-restarts if it crashes.  │
│                                                                 │
│    worker:                    →    ECS Fargate (separate service)│
│      command: celery worker        Same container image,         │
│                                    different command.            │
│                                    No public port needed.        │
│                                                                 │
│    beat:                      →    ECS Fargate (scheduled task)  │
│      command: celery beat          Runs your periodic scheduler. │
│                                                                 │
│    db:                        →    RDS PostgreSQL                │
│      image: postgres:15            AWS manages backups, patches, │
│      volumes: [pgdata:...]         failover, disk growth.        │
│                                    You just connect.             │
│                                                                 │
│    redis:                     →    ElastiCache Redis             │
│      image: redis:7                AWS manages memory, failover, │
│                                    replication.                  │
│                                                                 │
│  volumes:                                                       │
│    pgdata:                    →    RDS handles its own storage   │
│    uploads:                   →    S3 Bucket                     │
│                                    Virtually unlimited file      │
│                                    storage. Pay per GB.          │
│                                                                 │
│  networks:                                                      │
│    app-network:               →    VPC (Virtual Private Cloud)   │
│                                    Your own isolated network.    │
│                                    Firewall rules per service.   │
│                                                                 │
│  (implied)                                                      │
│    .env secrets               →    AWS Secrets Manager           │
│    health checks              →    ALB health checks             │
│    port 8000 exposed          →    Application Load Balancer     │
│                                    + Route 53 DNS                │
│                                    + TLS certificate             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "Look at this mapping. Your docker-compose.yml has 5 services and 2 volumes. In AWS, that becomes roughly 8-10 resources. Does the architecture change? No. The SAME containers run, the SAME database stores data, the SAME Redis caches results. The only thing that changes is WHO MANAGES THE HARDWARE underneath."

**The killer insight:**

> "Your Python code doesn't change. Not one line. Your `DATABASE_URL` used to point to `db:5432` (the Docker network hostname). Now it points to `my-rds-instance.abc123.us-east-1.rds.amazonaws.com:5432`. That's it. That's the whole migration. Your pydantic-settings configuration (Lecture 2) handles this automatically through environment variables."

---

## 2.4 Only Five Things Matter

```
┌─────────────────────────────────────────────────────────────────┐
│                   ONLY FIVE THINGS MATTER                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AWS has 200+ services. You need FIVE for your backend:         │
│                                                                 │
│                                                                 │
│                    ┌─────────────┐                              │
│                    │  COMPUTE    │  ← Where your code runs      │
│                    └──────┬──────┘                              │
│                           │                                     │
│             ┌─────────────┼─────────────┐                      │
│             │             │             │                        │
│             ▼             ▼             ▼                        │
│      ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│      │ DATABASE │  │ STORAGE  │  │ CACHE    │                   │
│      │ (RDS)    │  │ (S3)     │  │(Elasti-  │                   │
│      └──────────┘  └──────────┘  │ Cache)   │                   │
│                                  └──────────┘                   │
│                                                                 │
│      All of the above sit inside:                               │
│                                                                 │
│      ┌──────────────────────────────────────────┐               │
│      │            NETWORKING (VPC)               │               │
│      │  with IAM controlling who accesses what   │               │
│      └──────────────────────────────────────────┘               │
│                                                                 │
│                                                                 │
│  Everything else (200+ services) is either:                     │
│  • A variant of these five                                      │
│  • A convenience layer on top of these five                     │
│  • Something you don't need yet                                 │
│                                                                 │
│  Don't let the catalog overwhelm you.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: CORE SERVICES

## 3.1 Compute — Where Your Code Runs

**This is the most important category. Multiple options exist on a spectrum.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      COMPUTE OPTIONS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MORE CONTROL                                    LESS CONTROL   │
│  MORE WORK                                       LESS WORK      │
│  ◀─────────────────────────────────────────────────────────▶    │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │   EC2    │  │   ECS    │  │  ECS on  │  │  Lambda  │        │
│  │          │  │  on EC2  │  │ Fargate  │  │          │        │
│  │ Virtual  │  │          │  │          │  │ Function │        │
│  │ Machine  │  │ You run  │  │ AWS runs │  │ as a     │        │
│  │          │  │ Docker   │  │ Docker   │  │ Service  │        │
│  │ "Empty   │  │ on your  │  │ for you  │  │          │        │
│  │  house"  │  │ VMs      │  │          │  │ "Hotel"  │        │
│  │          │  │          │  │"Furnished│  │          │        │
│  │          │  │          │  │ apt"     │  │          │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│                                                                 │
│  You manage:   You manage:   You manage:    You manage:         │
│  • OS          • Containers  • Containers   • Code only         │
│  • Patches     • Task defs   • Task defs    • (a function)      │
│  • Docker      • Scaling     AWS manages:   AWS manages:        │
│  • Networking  AWS manages:  • Servers       • Everything       │
│  • Scaling     • Scheduling  • Scaling                          │
│  • Everything                • Patches                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### EC2 — The Virtual Machine

```
┌─────────────────────────────────────────────────────────────────┐
│                        EC2                                      │
│              Elastic Compute Cloud                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it is:                                                    │
│  A virtual computer in AWS's data center.                       │
│  You choose the CPU, RAM, and disk.                             │
│  You get SSH access. It's YOUR machine (rented).                │
│                                                                 │
│  Like:                                                          │
│  Renting an empty house. Four walls, power outlets,             │
│  water hookup. You install everything else.                     │
│                                                                 │
│                                                                 │
│  What you'd do to run your capstone on EC2:                     │
│                                                                 │
│    1. Launch an EC2 instance (pick size: t3.micro, t3.small)    │
│    2. SSH into it (ssh ec2-user@<public-ip>)                    │
│    3. Install Docker on it                                      │
│    4. Copy your docker-compose.yml                              │
│    5. Run docker compose up -d                                  │
│    6. Configure security group to allow port 443                │
│    7. Point your domain to the public IP                        │
│                                                                 │
│  Pros: Full control, cheapest per hour, simple mental model     │
│  Cons: YOU patch the OS. YOU restart if it crashes.             │
│        YOU handle scaling. It's a pet, not cattle.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Instance types — sizes of houses:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   EC2 INSTANCE TYPES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Type          vCPUs   RAM     Use Case           ~$/month      │
│  ──────────    ─────   ───     ────────           ────────      │
│  t3.micro      2       1 GB   Testing, free tier  ~$8           │
│  t3.small      2       2 GB   Small APIs          ~$15          │
│  t3.medium     2       4 GB   Production APIs     ~$30          │
│  t3.large      2       8 GB   Heavier workloads   ~$60          │
│  m5.xlarge     4       16 GB  Serious production  ~$140         │
│                                                                 │
│  The naming pattern:                                            │
│  ┌────────────────────────────────────────────┐                 │
│  │  t3.medium                                  │                 │
│  │  │  │                                       │                 │
│  │  │  └─ Generation (3rd gen of T family)     │                 │
│  │  │                                          │                 │
│  │  └─── Size (micro < small < medium < large) │                 │
│  │                                             │                 │
│  │  t = General purpose (burstable)            │                 │
│  │  m = General purpose (steady)               │                 │
│  │  c = Compute optimized (CPU-heavy)          │                 │
│  │  r = Memory optimized (RAM-heavy)           │                 │
│  └────────────────────────────────────────────┘                 │
│                                                                 │
│  For YOUR FastAPI backend:                                      │
│  t3.small or t3.medium is likely sufficient for early           │
│  production. You can always resize later.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### ECS Fargate — Run Your Container Without Managing Servers

```
┌─────────────────────────────────────────────────────────────────┐
│                    ECS FARGATE                                  │
│          "docker compose up, but in the cloud"                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it is:                                                    │
│  You give AWS your Docker image. AWS runs it.                   │
│  No SSH. No OS to patch. No server to manage.                   │
│  You define: which image, how much CPU/RAM, how many copies.    │
│  AWS handles: where it runs, keeping it alive, networking.      │
│                                                                 │
│  Like:                                                          │
│  A furnished apartment. Bring your clothes (container image),   │
│  the landlord handles the rest.                                 │
│                                                                 │
│                                                                 │
│  How it maps to your docker-compose:                            │
│                                                                 │
│  docker-compose.yml concept  │  ECS Fargate concept             │
│  ────────────────────────────│─────────────────────             │
│  service (api, worker, beat) │  ECS Service                     │
│  image: myapp:latest         │  Task Definition (image + config)│
│  ports: ["8000:8000"]        │  Load Balancer target             │
│  environment:                │  Task Definition env vars         │
│  depends_on:                 │  Service discovery / networking   │
│  deploy.replicas: 3          │  Desired count: 3                │
│                                                                 │
│                                                                 │
│  This is the closest thing to "docker compose in production."   │
│  It's the sweet spot for most small-to-medium backends.         │
│                                                                 │
│  Pros: No servers to manage, scales easily, pay per second      │
│  Cons: Slightly more expensive than raw EC2, less control       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The container registry — where your Docker image lives:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  YOUR CI/CD PIPELINE (Lecture 3)                                │
│                                                                 │
│  GitHub Push → GitHub Actions → Build Image → Push to ECR       │
│                                                  │              │
│                                                  ▼              │
│                                    ┌─────────────────────┐      │
│                                    │  ECR                 │      │
│                                    │  (Elastic Container  │      │
│                                    │   Registry)          │      │
│                                    │                      │      │
│                                    │  Stores your Docker  │      │
│                                    │  images. Like Docker  │     │
│                                    │  Hub, but private    │      │
│                                    │  and inside AWS.     │      │
│                                    └──────────┬──────────┘      │
│                                               │                 │
│                                               ▼                 │
│                                    ┌─────────────────────┐      │
│                                    │  ECS Fargate         │      │
│                                    │  Pulls latest image  │      │
│                                    │  Runs your container │      │
│                                    └─────────────────────┘      │
│                                                                 │
│  This is the "CD" part of your CI/CD pipeline from Lecture 3.   │
│  Now you know WHERE the pipeline deploys TO.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Database — Managed Persistence

**Connection to Week 5-7: You've been running Postgres locally. Here's the production version.**

```
┌─────────────────────────────────────────────────────────────────┐
│                          RDS                                    │
│              Relational Database Service                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it is:                                                    │
│  AWS runs PostgreSQL (or MySQL, MariaDB, etc.) FOR you.         │
│  Same Postgres you've been using since Week 5.                  │
│  Same SQL. Same everything. Different operator.                 │
│                                                                 │
│                                                                 │
│  What YOUR docker-compose does:       What RDS does:            │
│  ─────────────────────────────        ──────────────            │
│  Runs postgres:15 image         ✅    Runs PostgreSQL 15        │
│  Stores data in a volume        ✅    Stores on managed disk    │
│  ❌ No automatic backups         ✅    Daily automatic backups   │
│  ❌ No failover                  ✅    Multi-AZ failover option  │
│  ❌ No encryption at rest        ✅    Encryption at rest        │
│  ❌ No monitoring dashboard      ✅    CloudWatch metrics        │
│  ❌ You run Alembic manually     ~     You still run Alembic    │
│  ❌ Data lost if volume breaks   ✅    Point-in-time recovery   │
│  ❌ No security patches          ✅    Automated minor patches   │
│                                                                 │
│                                                                 │
│  The ONLY thing that changes in your code:                      │
│                                                                 │
│  # Local (docker-compose)                                       │
│  DATABASE_URL=postgresql+asyncpg://user:pass@db:5432/saas       │
│                                            ──                   │
│                                             └─ Docker hostname  │
│                                                                 │
│  # Production (RDS)                                             │
│  DATABASE_URL=postgresql+asyncpg://user:pass@my-rds.abc123      │
│    .us-east-1.rds.amazonaws.com:5432/saas                       │
│                                 ─────────────────────────       │
│                                  └─ RDS endpoint hostname       │
│                                                                 │
│  YOUR PYTHON CODE: Zero changes. SQLAlchemy doesn't care        │
│  whether Postgres runs in Docker or RDS. A connection           │
│  string is a connection string.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The pydantic-settings connection (Lecture 2 callback):**

```python
# Your settings class ALREADY handles this.
# This is why we taught pydantic-settings.

class Settings(BaseSettings):
    database_url: str  # ← Different value per environment

    model_config = SettingsConfigDict(env_file=".env")

# Local:  DATABASE_URL=postgresql+asyncpg://user:pass@db:5432/saas
# Prod:   DATABASE_URL=postgresql+asyncpg://user:pass@my-rds...5432/saas
# ^^^
# Same code. Different .env file. That's the entire migration.
```

**RDS sizing:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    RDS INSTANCE SIZES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Type            vCPUs  RAM     Storage   ~$/month              │
│  ────────────    ─────  ───     ───────   ────────              │
│  db.t3.micro     2      1 GB   20 GB     ~$15   (free tier!)   │
│  db.t3.small     2      2 GB   50 GB     ~$30                  │
│  db.t3.medium    2      4 GB   100 GB    ~$65                  │
│  db.r5.large     2      16 GB  500 GB    ~$175                 │
│                                                                 │
│  For YOUR capstone in early production:                         │
│  db.t3.micro or db.t3.small is plenty.                         │
│  You can resize later without downtime (mostly).                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**ElastiCache — managed Redis (same story, shorter version):**

```
┌─────────────────────────────────────────────────────────────────┐
│                      ELASTICACHE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Same Redis you used in Week 10.                                │
│  AWS manages memory, replication, failover.                     │
│                                                                 │
│  Your code change:                                              │
│                                                                 │
│  # Local                                                        │
│  REDIS_URL=redis://redis:6379                                   │
│                                                                 │
│  # Production                                                   │
│  REDIS_URL=redis://my-cluster.abc123.cache.amazonaws.com:6379   │
│                                                                 │
│  Same pattern. Same zero code changes.                          │
│  Your redis.asyncio client connects identically.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Storage — Objects and Files

**Connection to Week 13-14: Your capstone has file uploads. Where do files go in production?**

```
┌─────────────────────────────────────────────────────────────────┐
│                          S3                                     │
│                Simple Storage Service                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it is:                                                    │
│  A virtually unlimited key-value store for FILES.               │
│  You store objects (files) in buckets (top-level containers).   │
│  Each object has a unique key (like a file path).               │
│                                                                 │
│                                                                 │
│  ❌ S3 is NOT a filesystem.                                     │
│                                                                 │
│  There are no directories. No folders. No hierarchy.            │
│  "folders" in S3 are an illusion created by key prefixes.       │
│                                                                 │
│  What looks like:     /uploads/users/42/avatar.png              │
│  Is actually just:    A key string "uploads/users/42/avatar.png"│
│                       pointing to a blob of bytes.              │
│                                                                 │
│                                                                 │
│  S3 Concepts:                                                   │
│                                                                 │
│  ┌───────────────────────────────────────────────┐              │
│  │  BUCKET: my-saas-uploads                       │              │
│  │  (globally unique name, like a domain)         │              │
│  │                                                │              │
│  │  OBJECTS:                                      │              │
│  │  ├─ avatars/user-42.png     (2.1 MB)          │              │
│  │  ├─ avatars/user-99.png     (1.8 MB)          │              │
│  │  ├─ exports/report-jan.csv  (50 KB)           │              │
│  │  ├─ exports/report-feb.csv  (48 KB)           │              │
│  │  └─ attachments/task-7.pdf  (3.2 MB)          │              │
│  │                                                │              │
│  │  Each object has:                              │              │
│  │  • Key (the "path")                            │              │
│  │  • Value (the file bytes)                      │              │
│  │  • Metadata (content-type, timestamps)         │              │
│  │  • Access permissions                          │              │
│  └───────────────────────────────────────────────┘              │
│                                                                 │
│                                                                 │
│  Why S3 and not just a disk?                                    │
│                                                                 │
│  Local disk (EC2 volume):          S3:                           │
│  • Limited size (resize needed)    • Virtually unlimited         │
│  • One machine can access it       • Any service can access it  │
│  • If disk fails, data gone        • 99.999999999% durability   │
│  • You manage backups              • Automatic redundancy       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Presigned URLs — how your API handles file uploads in production:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   PRESIGNED URLs                                │
│         (Connection to Week 13-14 Capstone)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The problem:                                                   │
│  Users upload files. You don't want 100MB files flowing         │
│  THROUGH your FastAPI server. That wastes your server's         │
│  bandwidth and memory.                                          │
│                                                                 │
│  The solution: Presigned URLs                                   │
│                                                                 │
│  1. Client asks your API: "I want to upload avatar.png"         │
│                                                                 │
│  2. Your API generates a presigned URL:                         │
│     A temporary, time-limited URL that gives the client         │
│     permission to upload directly to S3.                        │
│                                                                 │
│  3. Client uploads directly to S3 (bypasses your server).       │
│                                                                 │
│  4. Client tells your API: "Upload complete, here's the key."  │
│                                                                 │
│  5. Your API stores the S3 key in the database.                 │
│                                                                 │
│                                                                 │
│  ┌────────┐          ┌──────────┐          ┌────────┐           │
│  │ Client │─── 1 ───▶│ Your API │          │   S3   │           │
│  │        │◀── 2 ────│          │─ signs ─▶│        │           │
│  │        │─── 3 ───────────────────upload─▶│        │           │
│  │        │─── 4 ───▶│          │          │        │           │
│  │        │          │  saves   │          │        │           │
│  │        │          │  key     │          │        │           │
│  └────────┘          └──────────┘          └────────┘           │
│                                                                 │
│  Your server never touches the file bytes.                      │
│  S3 handles the heavy lifting.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Networking — VPC Basics

**Connection to Week 9 (Security): This is the infrastructure-level security layer.**

```
┌─────────────────────────────────────────────────────────────────┐
│                        VPC                                      │
│               Virtual Private Cloud                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it is:                                                    │
│  Your own private, isolated network inside AWS.                 │
│  Nothing gets in or out unless you explicitly allow it.         │
│                                                                 │
│  Like:                                                          │
│  A gated community. Only residents (your services) are inside.  │
│  The gate (security group) decides who can enter from outside.  │
│  Internal roads (subnets) connect the buildings.                │
│                                                                 │
│                                                                 │
│  ┌─────────── YOUR VPC (10.0.0.0/16) ───────────────┐          │
│  │                                                    │          │
│  │  ┌─── PUBLIC SUBNET (10.0.1.0/24) ──────┐        │          │
│  │  │                                        │        │          │
│  │  │  ┌───────────┐     ┌───────────┐      │        │          │
│  │  │  │ Load      │     │ NAT       │      │        │          │
│  │  │  │ Balancer  │     │ Gateway   │      │        │          │
│  │  │  │ (ALB)     │     │           │      │        │          │
│  │  │  └─────┬─────┘     └─────┬─────┘      │        │          │
│  │  │        │                 │             │        │          │
│  │  └────────│─────────────────│─────────────┘        │          │
│  │           │                 │                      │          │
│  │  ┌────────│─────────────────│──────────────┐       │          │
│  │  │ PRIVATE│SUBNET (10.0.2.0│/24)          │       │          │
│  │  │        │                 │              │       │          │
│  │  │   ┌────▼────┐    ┌──────▼──────┐       │       │          │
│  │  │   │ ECS     │    │ ECS         │       │       │          │
│  │  │   │ Fargate │    │ Fargate     │       │       │          │
│  │  │   │ (API)   │    │ (Worker)    │       │       │          │
│  │  │   └────┬────┘    └─────────────┘       │       │          │
│  │  │        │                               │       │          │
│  │  └────────│───────────────────────────────┘       │          │
│  │           │                                       │          │
│  │  ┌────────│───────────────────────────────┐       │          │
│  │  │ PRIVATE│SUBNET (10.0.3.0/24)          │       │          │
│  │  │  (data │tier — most isolated)          │       │          │
│  │  │        │                               │       │          │
│  │  │   ┌────▼────┐    ┌────────────┐        │       │          │
│  │  │   │  RDS    │    │ ElastiCache│        │       │          │
│  │  │   │ Postgres│    │ Redis      │        │       │          │
│  │  │   └─────────┘    └────────────┘        │       │          │
│  │  │                                        │       │          │
│  │  │  ❌ NO internet access from here        │       │          │
│  │  │                                        │       │          │
│  │  └────────────────────────────────────────┘       │          │
│  │                                                    │          │
│  └────────────────────────────────────────────────────┘          │
│                                                                 │
│  INTERNET ←──▶ Load Balancer (public) ←──▶ API (private)        │
│                                            ←──▶ DB (private)    │
│                                                                 │
│  The database is NEVER exposed to the internet.                 │
│  Only your API can talk to it, through the private subnet.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Security Groups — the per-service firewall:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   SECURITY GROUPS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Security Groups are firewall rules attached to each resource.  │
│  They define: "Who can talk to me, on which port?"              │
│                                                                 │
│  Like: An apartment buzzer. You define who's allowed in.        │
│                                                                 │
│                                                                 │
│  LOAD BALANCER security group:                                  │
│  ┌────────────────────────────────────────┐                     │
│  │  INBOUND:                              │                     │
│  │  ✅ Port 443 (HTTPS) from 0.0.0.0/0   │  ← Anyone on        │
│  │  ✅ Port 80 (HTTP) from 0.0.0.0/0     │    the internet     │
│  │  ❌ Everything else: DENIED            │                     │
│  └────────────────────────────────────────┘                     │
│                                                                 │
│  API (ECS) security group:                                      │
│  ┌────────────────────────────────────────┐                     │
│  │  INBOUND:                              │                     │
│  │  ✅ Port 8000 from LOAD BALANCER SG    │  ← Only the LB     │
│  │  ❌ Everything else: DENIED            │    can reach it     │
│  └────────────────────────────────────────┘                     │
│                                                                 │
│  DATABASE (RDS) security group:                                 │
│  ┌────────────────────────────────────────┐                     │
│  │  INBOUND:                              │                     │
│  │  ✅ Port 5432 from API SG only         │  ← Only your       │
│  │  ❌ Everything else: DENIED            │    API can reach DB │
│  └────────────────────────────────────────┘                     │
│                                                                 │
│                                                                 │
│  Traffic flow:                                                  │
│  Internet → :443 → LB → :8000 → API → :5432 → DB              │
│                                                                 │
│  A hacker on the internet CANNOT directly reach your database.  │
│  They'd have to breach the load balancer, THEN the API,         │
│  THEN the DB security group. Defense in depth.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Remember CORS from Week 9? CORS is application-level access control — which DOMAINS can call your API. Security groups are network-level access control — which MACHINES can talk to which. They complement each other. CORS protects your API from unauthorized browser requests. Security groups protect your entire infrastructure from unauthorized network traffic."

---

## 3.5 IAM — The Permission System

**Connection to Week 9 (RBAC): IAM is RBAC, but for infrastructure.**

```
┌─────────────────────────────────────────────────────────────────┐
│                          IAM                                    │
│              Identity and Access Management                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In Week 9, you built RBAC for your SaaS:                       │
│  • Users have roles (admin, member, viewer)                     │
│  • Roles have permissions (can_create, can_delete)              │
│  • Endpoints check: "Does this user have permission?"           │
│                                                                 │
│  IAM is the SAME PATTERN, but for cloud resources:              │
│                                                                 │
│                                                                 │
│  YOUR APP (Week 9)              AWS IAM                         │
│  ─────────────────              ───────                         │
│  User (person)                  IAM User (person or service)    │
│  Role (admin, member)           IAM Role (set of permissions)   │
│  Permission (can_delete_task)   IAM Policy (can delete S3 obj)  │
│  "Is this user an admin?"       "Can this service access RDS?"  │
│                                                                 │
│                                                                 │
│  KEY PRINCIPLE: LEAST PRIVILEGE                                 │
│  ────────────────────────────                                   │
│  Every user, every service gets ONLY the permissions            │
│  it absolutely needs. Nothing more.                             │
│                                                                 │
│  Your API server:                                               │
│  ✅ Can read/write to RDS                                       │
│  ✅ Can read/write to ElastiCache                               │
│  ✅ Can put objects in S3                                        │
│  ❌ Cannot create/delete EC2 instances                          │
│  ❌ Cannot modify IAM policies                                  │
│  ❌ Cannot access billing information                           │
│                                                                 │
│  Your CI/CD pipeline:                                           │
│  ✅ Can push images to ECR                                      │
│  ✅ Can update ECS service (deploy new version)                  │
│  ❌ Cannot access the database directly                         │
│  ❌ Cannot modify VPC networking                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why IAM matters — the horror story:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   IAM HORROR STORY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Developer commits AWS access keys to public GitHub repo.       │
│  Bot finds the keys within MINUTES.                             │
│  Bot spins up 100 GPU instances for crypto mining.              │
│  Developer gets a $50,000 AWS bill the next morning.            │
│                                                                 │
│  This happens REGULARLY. It is not hypothetical.                │
│                                                                 │
│                                                                 │
│  RULES:                                                         │
│                                                                 │
│  1. NEVER commit AWS keys to git. Ever.                         │
│     (pydantic-settings + .env + .gitignore, from Lecture 2)     │
│                                                                 │
│  2. Use IAM ROLES for services, not access keys.                │
│     Your ECS task gets a role attached to it — no keys needed.  │
│                                                                 │
│  3. Enable MFA on your root account. Day one.                   │
│                                                                 │
│  4. Use the principle of least privilege. Always.               │
│                                                                 │
│  5. Rotate any credentials that DO exist regularly.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This is why we taught secrets management in Lecture 2. `pydantic-settings` reads from environment variables. `.env` is in `.gitignore`. Secrets Manager stores credentials encrypted. These aren't theoretical best practices — they prevent five-figure bills and data breaches."

---

# PART 4: THE REAL DECISIONS

## 4.1 The Responsibility Spectrum

**Every cloud service sits on a spectrum. Your job is to choose your position.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE RESPONSIBILITY SPECTRUM                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOU MANAGE                                       PROVIDER      │
│  EVERYTHING                                       MANAGES       │
│  ◀────────────────────────────────────────────────EVERYTHING──▶ │
│                                                                 │
│                                                                 │
│  On-Premises   IaaS         PaaS         FaaS     SaaS          │
│  ───────────   ────         ────         ────     ────          │
│  Your own      EC2          Elastic      Lambda   Gmail         │
│  hardware                   Beanstalk             Stripe        │
│                             Railway                             │
│                             Fly.io                              │
│                                                                 │
│                                                                 │
│  What YOU manage at each level:                                 │
│                                                                 │
│               On-Prem    IaaS    PaaS    FaaS    SaaS           │
│  ───────────────────────────────────────────────────            │
│  Application    You       You     You     You     ──            │
│  Data           You       You     You     You     ──            │
│  Runtime        You       You     ~~       ──     ──            │
│  OS             You       You      ──      ──     ──            │
│  Virtualize     You        ──      ──      ──     ──            │
│  Servers        You        ──      ──      ──     ──            │
│  Storage        You        ──      ──      ──     ──            │
│  Networking     You        ──      ──      ──     ──            │
│  Physical       You        ──      ──      ──     ──            │
│                                                                 │
│                 You = You are responsible                        │
│                  ── = Provider is responsible                    │
│                  ~~ = Shared (you configure, they maintain)      │
│                                                                 │
│                                                                 │
│  The further right you go:                                      │
│  • Less control                                                 │
│  • Less ops work                                                │
│  • Often higher cost per unit                                   │
│  • Faster to start                                              │
│  • Harder to customize                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "When you deployed to Railway or Fly.io in Lecture 3 — where were you on this spectrum?"

Answer: **PaaS.** You pushed code or a Docker image. They handled the OS, runtime, networking, TLS, scaling. You managed your application and data. That's PaaS.

> "And when you were running `docker compose up` locally — where were you?"

Answer: **You were the cloud.** You managed everything: the OS (your laptop), the runtime (Docker), the networking (Docker networks), the storage (Docker volumes). The only difference between you and AWS is that you're one person with one machine, and they're thousands of people with millions of machines.

---

## 4.2 Managed vs Self-Hosted (Three Case Studies)

**Case Study 1: Your PostgreSQL**

```
┌─────────────────────────────────────────────────────────────────┐
│          SELF-HOSTED vs MANAGED: PostgreSQL                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION A: Postgres in Docker on EC2 (self-hosted)              │
│  ─────────────────────────────────────────────────              │
│  You:                                                           │
│  • Install Docker on EC2                                        │
│  • Run postgres:15 container                                    │
│  • Set up a cron job for pg_dump backups                        │
│  • Monitor disk space manually                                  │
│  • Apply Postgres security patches yourself                     │
│  • Pray the EBS volume doesn't corrupt                          │
│  • Handle failover yourself (spoiler: you won't)                │
│                                                                 │
│  Cost: ~$15/month (t3.micro EC2 + EBS volume)                   │
│  Ops burden: HIGH. You are the DBA.                             │
│                                                                 │
│                                                                 │
│  OPTION B: RDS PostgreSQL (managed)                             │
│  ──────────────────────────────────                             │
│  AWS:                                                           │
│  • Runs Postgres 15 for you                                     │
│  • Automated daily backups with point-in-time recovery          │
│  • Automated minor version patches                              │
│  • Multi-AZ failover (optional — automatic recovery)            │
│  • Encrypted storage                                            │
│  • Monitoring built in                                          │
│  • Disk auto-scales when needed                                 │
│                                                                 │
│  Cost: ~$15-30/month (db.t3.micro — free tier eligible!)        │
│  Ops burden: LOW. You connect and use it.                       │
│                                                                 │
│                                                                 │
│  THE QUESTION:                                                  │
│  Is the $0-15/month savings worth being the on-call DBA         │
│  at 3 AM when your disk fills up?                               │
│                                                                 │
│  For almost every team: NO. Use RDS.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Case Study 2: Your Redis**

```
┌─────────────────────────────────────────────────────────────────┐
│          SELF-HOSTED vs MANAGED: Redis                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION A: Redis in Docker on EC2 (self-hosted)                 │
│                                                                 │
│  • Similar ops burden as self-hosted Postgres                   │
│  • Redis is simpler to operate than Postgres                    │
│  • But: no automatic failover, no replication out of the box    │
│  • If it crashes, your cache is cold and your app slows down    │
│                                                                 │
│  Cost: ~$0 (run on same EC2 as your app)                        │
│  Risk: Single point of failure                                  │
│                                                                 │
│                                                                 │
│  OPTION B: ElastiCache Redis (managed)                          │
│                                                                 │
│  • Automatic failover with replicas                             │
│  • Managed patching                                             │
│  • Monitoring, automatic snapshots                              │
│                                                                 │
│  Cost: ~$12-25/month (cache.t3.micro)                           │
│                                                                 │
│                                                                 │
│  THE NUANCE:                                                    │
│  Remember from Week 10 — your app should handle Redis being     │
│  unavailable (graceful degradation). If your cache layer is     │
│  truly optional and you're small, self-hosted Redis on the      │
│  same EC2 is a reasonable early-stage choice.                   │
│                                                                 │
│  Decision depends on: Is cache critical or nice-to-have?        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Case Study 3: Your FastAPI Application**

```
┌─────────────────────────────────────────────────────────────────┐
│          COMPUTE OPTIONS FOR YOUR FASTAPI APP                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION A: EC2 + Docker Compose                                 │
│  ───────────────────────────────                                │
│  Literally what you do locally, but on a rented server.         │
│  SSH in, docker compose up. Done.                               │
│                                                                 │
│  ✅ Simplest to understand (you've been doing this)             │
│  ✅ Cheapest ($8-30/month)                                      │
│  ❌ You manage uptime, deploys, scaling                         │
│  ❌ Zero-downtime deploys require manual work                   │
│                                                                 │
│  Good for: Hobby projects, MVPs, learning                       │
│                                                                 │
│                                                                 │
│  OPTION B: ECS Fargate                                          │
│  ─────────────────────                                          │
│  Give AWS your Docker image. It runs containers for you.        │
│                                                                 │
│  ✅ No servers to manage                                        │
│  ✅ Built-in load balancing, health checks                      │
│  ✅ Scales by adding more containers                            │
│  ❌ More setup complexity initially                             │
│  ❌ Slightly more expensive (~$30-60/month)                     │
│                                                                 │
│  Good for: Production apps with real users                      │
│                                                                 │
│                                                                 │
│  OPTION C: Railway / Fly.io (PaaS)                              │
│  ─────────────────────────────────                              │
│  Push code or Docker image. They handle everything.             │
│                                                                 │
│  ✅ Fastest to ship (minutes, not hours)                        │
│  ✅ Zero infrastructure knowledge needed                        │
│  ❌ Least control                                               │
│  ❌ Most expensive at scale (~$5-20/month early, grows fast)    │
│                                                                 │
│  Good for: Prototypes, demos, small projects                    │
│                                                                 │
│                                                                 │
│  OPTION D: Lambda (Serverless) — we'll cover in Part 5          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The common startup path:**

```
┌─────────────────────────────────────────────────────────────────┐
│               TYPICAL EVOLUTION                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Day 1:   Railway/Fly.io. Ship fast. Validate the idea.         │
│           Don't think about infrastructure.                     │
│           │                                                     │
│           ▼                                                     │
│  Month 3: Getting users. PaaS bill climbing. Need more          │
│           control. Migrate to ECS Fargate + RDS.                │
│           │                                                     │
│           ▼                                                     │
│  Year 1:  Serious traffic. Need fine-tuning. Maybe              │
│           EC2 with auto-scaling groups. Or Kubernetes.           │
│           │                                                     │
│           ▼                                                     │
│  Year 3+: Multi-region. Dedicated infrastructure team.          │
│           Complex orchestration.                                │
│                                                                 │
│                                                                 │
│  Don't build for Year 3 on Day 1.                               │
│  Start simple. Migrate when the pain justifies the complexity.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Infrastructure as Code (Terraform Awareness)

**The problem first:**

> "Imagine you spent an afternoon clicking through the AWS console: created a VPC, configured subnets, launched an RDS instance, set up security groups, deployed ECS services. Everything works. Now your boss says: 'Set up an identical environment for staging.' What do you do?"

> "Do it all again, from memory, clicking through 50 screens, hoping you don't miss a setting? That's called ClickOps. And it's how production outages happen."

```
┌─────────────────────────────────────────────────────────────────┐
│              CLICKOPS vs INFRASTRUCTURE AS CODE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLICKOPS (Manual Console)          INFRASTRUCTURE AS CODE      │
│  ─────────────────────────          ──────────────────────      │
│  Click through AWS console          Write code that describes   │
│  to create resources                 the resources you want     │
│                                                                 │
│  ❌ Not repeatable                  ✅ Perfectly repeatable     │
│  ❌ No history / audit trail        ✅ Version controlled (git) │
│  ❌ "What did I configure?"         ✅ Read the code            │
│  ❌ Can't code review infra         ✅ PR review for infra      │
│  ❌ Staging ≠ Production            ✅ Same code, different vars│
│  ❌ Bus factor = 1                  ✅ Anyone can understand    │
│                                                                 │
│                                                                 │
│  IaC is to infrastructure what version control is to code.      │
│  You'd never edit production code directly on the server.       │
│  Don't edit production infrastructure directly in the console.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Terraform mental model:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   TERRAFORM                                     │
│            (by HashiCorp — the most popular IaC tool)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it is:                                                    │
│  You write files that DECLARE what infrastructure you want.     │
│  Terraform figures out how to create it.                        │
│                                                                 │
│  The mental model:                                              │
│                                                                 │
│  ┌────────────────────┐        ┌────────────────────┐           │
│  │  What you WROTE    │        │  What ACTUALLY      │           │
│  │  (desired state)   │        │  EXISTS in AWS      │           │
│  │                    │        │  (current state)    │           │
│  │  • 1 VPC           │        │  • 1 VPC            │           │
│  │  • 2 subnets       │  ───▶  │  • 2 subnets        │           │
│  │  • 1 RDS instance  │ apply  │  • 1 RDS instance   │           │
│  │  • 1 ECS service   │        │  • 1 ECS service    │           │
│  └────────────────────┘        └────────────────────┘           │
│                                                                 │
│  terraform plan  = "Here's what I WOULD change"                 │
│  terraform apply = "Make it so"                                 │
│                                                                 │
│  If you add a new resource to the file, Terraform creates it.   │
│  If you remove a resource, Terraform deletes it.                │
│  If you change a setting, Terraform updates it.                 │
│  Terraform tracks what exists in a STATE FILE.                  │
│                                                                 │
│                                                                 │
│  Think of it like a DECLARATIVE approach:                       │
│                                                                 │
│  Pydantic:   "I want data shaped like THIS."                    │
│              Pydantic validates/coerces it into that shape.      │
│                                                                 │
│  Terraform:  "I want infrastructure shaped like THIS."          │
│              Terraform provisions/updates it into that shape.    │
│                                                                 │
│  Same idea. Declare what. Let the tool figure out how.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Reading Terraform (Literacy, Not Fluency)

**You don't need to WRITE Terraform right now. You need to READ it.**

**You will encounter Terraform files at your first job. You need to know what you're looking at.**

```hcl
# main.tf — Terraform configuration for your capstone project
# DON'T memorize the syntax. Understand the INTENT.

# "I want a VPC"
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "saas-platform-vpc"
  }
}

# "I want a managed PostgreSQL database"
resource "aws_db_instance" "main" {
  engine         = "postgres"
  engine_version = "15"
  instance_class = "db.t3.micro"

  db_name  = "saas"
  username = "admin"
  password = var.db_password    # ← Variable, not hardcoded!

  allocated_storage = 20        # GB
  storage_encrypted = true

  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = 7   # Days of backups to keep
  skip_final_snapshot     = false

  tags = {
    Name = "saas-platform-db"
  }
}
```

**Reading it like a student:**

```
┌─────────────────────────────────────────────────────────────────┐
│                READING TERRAFORM                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  resource "aws_db_instance" "main" {                            │
│  ────────  ───────────────  ────                                │
│     │           │             │                                  │
│     │           │             └─ YOUR name for it (like a var)  │
│     │           └─────────────── AWS resource TYPE              │
│     └─────────────────────────── "I want a resource"            │
│                                                                 │
│                                                                 │
│  engine         = "postgres"    # What DB engine                │
│  engine_version = "15"          # Which version                 │
│  instance_class = "db.t3.micro" # How big (CPU/RAM)             │
│  db_name        = "saas"        # Database name                 │
│  password       = var.db_password  # From a VARIABLE (secret)   │
│                                                                 │
│  vpc_security_group_ids = [aws_security_group.db.id]            │
│                            ──────────────────────               │
│                            Reference to ANOTHER resource        │
│                            Terraform links them automatically   │
│                                                                 │
│                                                                 │
│  You can read this:                                             │
│  "Create a PostgreSQL 15 database, micro size, 20GB, encrypted, │
│   in the VPC we defined, with 7-day backup retention."          │
│                                                                 │
│  That's it. You just read Terraform.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The Terraform workflow (connection to Lecture 3 — CI/CD):**

```
┌─────────────────────────────────────────────────────────────────┐
│              TERRAFORM IN YOUR CI/CD PIPELINE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Remember your GitHub Actions pipeline from Lecture 3:          │
│                                                                 │
│  Push code → Lint → Test → Build image → Push → Deploy          │
│                                                                 │
│  With Terraform, infrastructure changes follow the SAME flow:   │
│                                                                 │
│  Push .tf files → terraform plan → Code review → terraform apply│
│                                                                 │
│                                                                 │
│  Developer:   "I need a bigger database."                       │
│  Change:      instance_class = "db.t3.micro"                    │
│               → instance_class = "db.t3.medium"                 │
│  PR review:   Team reviews the change.                          │
│  CI:          terraform plan shows: "Will modify 1 resource."   │
│  Merge:       terraform apply resizes the database.             │
│                                                                 │
│  Infrastructure changes are now:                                │
│  ✅ Version controlled                                          │
│  ✅ Code reviewed                                               │
│  ✅ Tested before applying                                      │
│  ✅ Auditable (who changed what, when, why)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You won't be writing Terraform in this course. But at your first job, you'll open a `main.tf` file and need to understand: what does our infrastructure look like? Now you can read it."

---

# PART 5: COST AWARENESS & SERVERLESS

## 5.1 How Cloud Billing Actually Works

**The utility bill analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 CLOUD BILLING MODEL                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cloud billing works like utility bills:                        │
│                                                                 │
│  ELECTRICITY (Compute)                                          │
│  ─────────────────────                                          │
│  Your EC2 instance is like leaving the lights on.               │
│  Running 24/7 = paying 24/7.                                    │
│  Turned off = no charge.                                        │
│  Billed per second (most services) or per hour.                 │
│                                                                 │
│  WATER (Data Transfer)                                          │
│  ─────────────────────                                          │
│  Data flowing IN to AWS: FREE (they want your data inside).     │
│  Data flowing OUT of AWS: COSTS MONEY (they want to keep it).   │
│  This is called "egress" and it surprises everyone.             │
│                                                                 │
│  STORAGE (Rent on your Storage Unit)                            │
│  ────────────────────────────────────                           │
│  S3: Pay per GB stored per month.                               │
│  RDS: Pay for the allocated storage.                            │
│  Data at rest = ongoing monthly cost.                           │
│                                                                 │
│  APPLIANCE RENTAL (Managed Services)                            │
│  ────────────────────────────────────                           │
│  RDS instance: Pay per hour, regardless of queries.             │
│  ElastiCache: Pay per hour, regardless of cache hits.           │
│  Like renting a dishwasher — you pay even if you don't use it.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A realistic monthly bill for your capstone:**

```
┌─────────────────────────────────────────────────────────────────┐
│         REALISTIC MONTHLY BILL: YOUR SAAS CAPSTONE              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MINIMAL SETUP (hobby/learning):                                │
│  ──────────────────────────────                                 │
│  ECS Fargate (API, 0.25 vCPU, 0.5GB)      ~$9/month            │
│  ECS Fargate (Worker, 0.25 vCPU, 0.5GB)   ~$9/month            │
│  RDS PostgreSQL (db.t3.micro)              ~$15/month           │
│  ElastiCache (cache.t3.micro)              ~$12/month           │
│  Application Load Balancer                 ~$16/month           │
│  S3 (1 GB storage)                         ~$0.02/month         │
│  Data transfer (10 GB out)                 ~$0.90/month         │
│  Route 53 (DNS)                            ~$0.50/month         │
│  ─────────────────────────────────────────────────              │
│  TOTAL                                     ~$62/month           │
│                                                                 │
│                                                                 │
│  vs Railway (PaaS equivalent):                                  │
│  Pro plan: $5/user/month + usage                                │
│  Similar setup: ~$30-50/month (but with less control)           │
│                                                                 │
│                                                                 │
│  vs AWS Free Tier (first 12 months):                            │
│  EC2 t3.micro: 750 hours/month free                             │
│  RDS db.t3.micro: 750 hours/month free                          │
│  S3: 5 GB free                                                  │
│  ─────────────────────────────────────────────────              │
│  TOTAL (free tier)                         ~$16-20/month        │
│  (ALB and data transfer aren't fully covered)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 The Cost Traps That Burn Beginners

```
┌─────────────────────────────────────────────────────────────────┐
│                    COST TRAPS ⚠️                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  TRAP 1: FORGETTING TO TURN THINGS OFF                          │
│  ─────────────────────────────────────                          │
│  You spin up an EC2 instance to experiment.                     │
│  You forget about it for 3 months.                              │
│  That's $24-90 you didn't mean to spend.                        │
│                                                                 │
│  RULE: After every experiment session, check the console.       │
│        Set up billing alerts ($5, $20, $50 thresholds).         │
│                                                                 │
│                                                                 │
│  TRAP 2: DATA TRANSFER (EGRESS)                                 │
│  ──────────────────────────────                                 │
│  Data INTO AWS: Free.                                           │
│  Data OUT of AWS: $0.09/GB.                                     │
│  Sounds cheap? Stream 1 TB of API responses: $92.                │
│  Serve 10 TB of S3 files (images, videos): $920.               │
│                                                                 │
│  RULE: Put a CDN (CloudFront) in front of S3 for public files. │
│        CDN egress is cheaper and cached.                        │
│                                                                 │
│                                                                 │
│  TRAP 3: NAT GATEWAY                                            │
│  ────────────────────                                           │
│  Your private subnet needs internet access (to pull Docker      │
│  images, call external APIs). NAT Gateway provides this.        │
│  It costs $0.045/hour + $0.045/GB processed.                    │
│  Running 24/7 = ~$32/month BEFORE data charges.                 │
│                                                                 │
│  RULE: Be aware this exists on your bill. It adds up silently.  │
│                                                                 │
│                                                                 │
│  TRAP 4: LOAD BALANCER MINIMUM                                  │
│  ─────────────────────────────                                  │
│  ALB costs ~$16/month even with zero traffic.                   │
│  It's a fixed cost. Budget for it.                              │
│                                                                 │
│                                                                 │
│  TRAP 5: MULTI-AZ DOUBLES COST                                  │
│  ─────────────────────────────                                  │
│  RDS Multi-AZ (auto-failover) = 2× the instance cost.          │
│  ElastiCache with replica = 2× the instance cost.              │
│  Worth it for production. Not for development.                  │
│                                                                 │
│  RULE: Have separate configs for dev and prod environments.     │
│        (pydantic-settings already supports this!)               │
│                                                                 │
│                                                                 │
│  ⚠️ SET UP BILLING ALERTS ON DAY ONE ⚠️                        │
│                                                                 │
│  AWS Console → Billing → Budgets → Create budget                │
│  Alert at: $10, $25, $50, $100                                  │
│  This is free and takes 2 minutes. Do it.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Serverless — Functions as a Service

```
┌─────────────────────────────────────────────────────────────────┐
│                      SERVERLESS                                 │
│               (Lambda / Cloud Functions)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it is:                                                    │
│  You write a SINGLE FUNCTION. Upload it.                        │
│  Cloud runs it ONLY when triggered.                             │
│  No server. No container. No EC2.                               │
│  You pay ONLY for the milliseconds it runs.                     │
│                                                                 │
│  Like:                                                          │
│  A hotel room. You don't rent by the month.                     │
│  You check in, use it, check out, pay per night.               │
│  When you're not there, you pay nothing.                        │
│                                                                 │
│                                                                 │
│  A Lambda function that handles a webhook:                      │
│                                                                 │
│  ┌──────────────────────────────────────┐                       │
│  │                                      │                       │
│  │  Event (HTTP request, S3 upload,     │                       │
│  │  schedule, queue message)            │                       │
│  │            │                         │                       │
│  │            ▼                         │                       │
│  │  ┌──────────────────┐               │                       │
│  │  │  Lambda Function  │               │                       │
│  │  │                   │               │                       │
│  │  │  • Starts in ~ms  │               │                       │
│  │  │  • Runs your code │               │                       │
│  │  │  • Returns result │               │                       │
│  │  │  • Shuts down     │               │                       │
│  │  └──────────────────┘               │                       │
│  │            │                         │                       │
│  │            ▼                         │                       │
│  │  Response (or nothing)               │                       │
│  │                                      │                       │
│  │  Cost: $0.0000002 per request        │                       │
│  │      + $0.0000167 per GB-second      │                       │
│  │                                      │                       │
│  │  1 million requests = ~$0.20         │                       │
│  │                                      │                       │
│  └──────────────────────────────────────┘                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What a Lambda function looks like in Python:**

```python
# lambda_function.py
# This is a COMPLETE Lambda function. That's it. The whole file.

import json

def handler(event, context):
    """
    AWS calls this function when triggered.
    
    event:   The incoming data (HTTP request, S3 event, etc.)
    context: Metadata about the invocation (timeout, request ID)
    """
    # event contains the trigger data
    body = json.loads(event.get("body", "{}"))
    
    user_name = body.get("name", "World")
    
    return {
        "statusCode": 200,
        "body": json.dumps({"message": f"Hello, {user_name}!"})
    }

# No FastAPI. No uvicorn. No server.
# AWS invokes handler() directly when a request arrives.
```

> "Notice: this looks like a regular Python function. That's because it IS a regular Python function. Lambda just calls it for you when an event occurs. No ASGI server, no Docker, no process management. You upload the function, AWS runs it."

---

## 5.4 When Serverless Fits (and When It Doesn't)

```
┌─────────────────────────────────────────────────────────────────┐
│                 SERVERLESS: WHEN & WHEN NOT                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ SERVERLESS SHINES:                                          │
│  ─────────────────────                                          │
│                                                                 │
│  • Webhook handlers                                             │
│    "When Stripe sends a payment event, run this function."      │
│    Traffic: Bursty. Idle 99% of the time.                       │
│                                                                 │
│  • Scheduled tasks                                              │
│    "Every night at 2 AM, generate a report."                    │
│    (Alternative to Celery Beat for simple tasks)                │
│                                                                 │
│  • Image/file processing on upload                              │
│    "When a file lands in S3, generate a thumbnail."             │
│                                                                 │
│  • Low-traffic APIs                                             │
│    An internal tool used 50 times a day.                        │
│    Why pay for a server running 24/7?                           │
│                                                                 │
│  Common pattern: $0.00 cost for weeks, then $0.03               │
│  when it actually runs.                                         │
│                                                                 │
│                                                                 │
│  ❌ SERVERLESS STRUGGLES:                                       │
│  ────────────────────────                                       │
│                                                                 │
│  • Your FastAPI app with persistent connections                  │
│    WebSockets (Week 12)? Lambda can't hold them.                │
│    Database connection pools? Lambda creates new                 │
│    connections per invocation — pool is meaningless.             │
│                                                                 │
│  • Long-running processes                                       │
│    Lambda has a 15-minute timeout. Your Celery tasks             │
│    that process hours of data? Not a fit.                       │
│                                                                 │
│  • Consistent high traffic                                      │
│    1000 requests/second, 24/7?                                  │
│    Lambda per-request pricing adds up FAST.                     │
│    A container running 24/7 is much cheaper here.               │
│                                                                 │
│  • Cold starts                                                  │
│    First invocation after idle = 100ms-1s startup.              │
│    For latency-sensitive APIs, this can be a problem.           │
│                                                                 │
│                                                                 │
│  YOUR CAPSTONE?                                                 │
│  It has WebSockets, background workers, connection pools,       │
│  persistent sessions. Lambda is NOT the right fit.              │
│  ECS Fargate or a PaaS is the right choice.                     │
│                                                                 │
│  But that webhook endpoint from Week 8? That could be Lambda.   │
│  Not every endpoint needs the same infrastructure.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Your Deployment Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│             DEPLOYMENT DECISION FRAMEWORK                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                       START HERE                                │
│                           │                                     │
│                           ▼                                     │
│                ┌─────────────────────┐                          │
│                │  How much time do   │                          │
│                │  you have?          │                          │
│                └──────────┬──────────┘                          │
│                     │          │                                │
│                  <1 day      >1 week                            │
│                     │          │                                │
│                     ▼          ▼                                │
│              ┌───────────┐  ┌─────────────────────┐            │
│              │ PaaS      │  │ Do you need         │            │
│              │ Railway   │  │ WebSockets or       │            │
│              │ Fly.io    │  │ background workers? │            │
│              │ Render    │  └──────────┬──────────┘            │
│              └───────────┘       │          │                   │
│                                 YES         NO                  │
│                                  │          │                   │
│                                  ▼          ▼                   │
│                          ┌───────────┐  ┌───────────┐          │
│                          │ Container │  │ Does it   │          │
│                          │ ECS       │  │ run only  │          │
│                          │ Fargate   │  │ on events?│          │
│                          │ Cloud Run │  └─────┬─────┘          │
│                          └───────────┘    │       │            │
│                                          YES      NO           │
│                                           │       │            │
│                                           ▼       ▼            │
│                                    ┌──────────┐ ┌──────────┐   │
│                                    │ Lambda / │ │ Container│   │
│                                    │ Cloud    │ │ or PaaS  │   │
│                                    │ Functions│ │          │   │
│                                    └──────────┘ └──────────┘   │
│                                                                 │
│                                                                 │
│  FOR YOUR CAPSTONE DELIVERABLE:                                 │
│                                                                 │
│  Quick demo / grading?     → Railway or Fly.io (PaaS)          │
│  Learning experience?      → ECS Fargate + RDS (IaaS)          │
│  Minimal budget?           → EC2 + Docker Compose               │
│                                                                 │
│  All three are valid. Choose based on YOUR goals.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Misconceptions

### Misconception 1: "The cloud is more secure by default"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ WRONG: "AWS is secure, so my stuff is secure."              │
│                                                                 │
│  ✅ REALITY: Shared responsibility model.                       │
│                                                                 │
│  AWS secures:                YOUR responsibility:               │
│  • Physical data centers     • Your application code            │
│  • Host operating system     • Your security groups config      │
│  • Network infrastructure    • Your IAM permissions             │
│  • Hypervisor                • Your data encryption             │
│                              • Your access keys                 │
│                              • Your database passwords          │
│                                                                 │
│  A misconfigured S3 bucket with public access?                  │
│  That's YOUR fault. AWS gave you the lock —                     │
│  you chose not to use it.                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Misconception 2: "Cloud is always cheaper"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ WRONG: "Moving to cloud will save money."                   │
│                                                                 │
│  ✅ REALITY: Cloud is cheaper for VARIABLE loads.               │
│              On-premises is cheaper for CONSTANT loads.          │
│                                                                 │
│  Startup with 100 users → Cloud wins.                           │
│  Enterprise with 10,000 constant users → Maybe not.             │
│                                                                 │
│  Cloud value isn't just cost. It's:                              │
│  • Speed of setup (minutes, not weeks)                          │
│  • Elasticity (scale up for Black Friday, scale down after)     │
│  • Reduced ops burden (no hardware to manage)                   │
│  • Global distribution (deploy to 20+ regions)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Misconception 3: "I need to learn all 200+ AWS services"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ WRONG: "AWS has 200+ services. I'll never learn enough."    │
│                                                                 │
│  ✅ REALITY: 90% of backends use fewer than 10 services.        │
│                                                                 │
│  The five we covered today (Compute, Database, Storage,         │
│  Networking, IAM) will carry you through your first             │
│  2-3 years of backend work.                                     │
│                                                                 │
│  Add a few convenience services:                                │
│  • CloudWatch (monitoring) — like your structlog, but hosted    │
│  • Route 53 (DNS) — points your domain to your resources        │
│  • ACM (certificates) — free TLS certs for HTTPS               │
│  • Secrets Manager — like your .env, but encrypted and shared   │
│                                                                 │
│  That's ~10 services. You're covered.                           │
│  Learn new ones when a specific problem demands it.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Misconception 4: "Serverless replaces containers"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ WRONG: "Lambda is the future. Containers are obsolete."     │
│                                                                 │
│  ✅ REALITY: They solve different problems.                     │
│                                                                 │
│  Lambda: Event-driven, stateless, short-lived tasks             │
│  Container: Long-running, stateful, persistent connections      │
│                                                                 │
│  Your FastAPI server? Container.                                │
│  Your webhook handler? Could be Lambda.                         │
│  Your Celery worker? Container.                                 │
│  Your nightly report cron? Could be Lambda.                     │
│                                                                 │
│  Most production systems use BOTH.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                  CLOUD QUICK REFERENCE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE CONCEPT:                                                   │
│      Cloud = rented infrastructure, billed by usage.            │
│      Your code doesn't change. The config does.                 │
│                                                                 │
│  CORE SERVICES MAP:                                             │
│      Compute:     ECS Fargate (run your Docker container)       │
│      Database:    RDS (managed Postgres — same SQL)             │
│      Cache:       ElastiCache (managed Redis — same commands)   │
│      Storage:     S3 (unlimited file storage)                   │
│      Network:     VPC (private network + security groups)       │
│      Permissions: IAM (RBAC for infrastructure)                 │
│                                                                 │
│  MANAGED vs SELF-HOSTED:                                        │
│      Managed = pay more, operate less (RDS, ElastiCache)        │
│      Self-hosted = pay less, operate more (Docker on EC2)       │
│      Default choice: Managed, unless budget forces otherwise.   │
│                                                                 │
│  INFRASTRUCTURE AS CODE:                                        │
│      Terraform = declare infrastructure in code, version it.    │
│      terraform plan  = preview changes                          │
│      terraform apply = make changes                             │
│      ClickOps is tech debt. Code your infra.                    │
│                                                                 │
│  COST RULES:                                                    │
│      1. Set billing alerts IMMEDIATELY                          │
│      2. Turn off what you're not using                          │
│      3. Egress (data out) costs real money                      │
│      4. Use free tier for learning (12 months on AWS)           │
│      5. Different configs for dev vs prod                       │
│                                                                 │
│  SERVERLESS:                                                    │
│      Lambda = function that runs on events, pay per invocation  │
│      Good for: webhooks, cron, event processing                 │
│      Bad for: WebSockets, long tasks, high constant traffic     │
│                                                                 │
│  GOLDEN RULE:                                                   │
│      Start simple. Migrate when the pain justifies complexity.  │
│      PaaS → Containers → VMs → Kubernetes                       │
│      Most apps never need to go past step 2.                    │
│                                                                 │
│  PROVIDER NAMES DON'T MATTER:                                   │
│      Learn concepts, not brands. EC2 = Compute Engine = Azure   │
│      VM. Same idea. Different name. Transferable knowledge.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CLOUD = YOUR DOCKER COMPOSE, RUN BY SOMEONE ELSE              │
│                                                                 │
│  ┌──────────────────┐          ┌──────────────────┐             │
│  │  YOUR LAPTOP     │          │  AWS / GCP       │             │
│  │                  │          │                  │             │
│  │  docker-compose  │ ──────▶  │  ECS + RDS +     │             │
│  │  up              │  same    │  ElastiCache +   │             │
│  │                  │  app,    │  S3 + VPC        │             │
│  │  localhost:8000  │  new     │                  │             │
│  │                  │  address │  myapp.com       │             │
│  └──────────────────┘          └──────────────────┘             │
│                                                                 │
│                                                                 │
│  WHAT CHANGES:                     WHAT STAYS THE SAME:         │
│  ─────────────                     ─────────────────────        │
│  • Where it runs                   • Your Python code           │
│  • Connection strings              • Your Docker image          │
│  • Who manages the hardware        • Your Alembic migrations    │
│  • How you pay for it              • Your test suite            │
│  • Scale and availability          • Your API contracts         │
│                                                                 │
│                                                                 │
│  THE HOUSING ANALOGY:                                           │
│  ├─ Parents' house = local dev (free, can't scale)              │
│  ├─ Empty house    = EC2 (full control, full responsibility)    │
│  ├─ Furnished apt  = Managed services (less work, more cost)    │
│  ├─ Hotel          = PaaS / Serverless (show up and use it)     │
│  └─ Pick the housing that fits your stage of life.              │
│                                                                 │
│                                                                 │
│  You are not learning "AWS."                                    │
│  You are learning WHERE SOFTWARE RUNS and HOW TO CHOOSE.        │
│  The provider is interchangeable. The thinking is permanent.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Lectures

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 15 DELIVERABLE (this week):                               │
│  └─ Deploy your capstone to a cloud platform.                   │
│     You now understand WHAT Railway/Fly.io abstract away,       │
│     and WHAT AWS services you'd use if doing it yourself.       │
│     Choose your deployment target with informed confidence.     │
│                                                                 │
│  WEEK 16, LECTURE 1 — System Design Fundamentals:               │
│  └─ Load balancers, horizontal scaling, database replicas.      │
│     These are all CLOUD SERVICES you learned today.             │
│     ALB = load balancer. RDS read replica = database scaling.   │
│     System design is just picking the right cloud services      │
│     and connecting them correctly.                              │
│                                                                 │
│  WEEK 16, LECTURE 2 — Architecture Patterns:                    │
│  └─ Monolith vs microservices? Each service = an ECS service.   │
│     Message queues? SQS (AWS), Cloud Tasks (GCP).               │
│     API gateway? AWS API Gateway or your ALB.                   │
│     Today's mental model is the foundation for all of this.     │
│                                                                 │
│  WEEK 16, LECTURE 3 — Interview Prep:                           │
│  └─ System design interviews expect you to NAME cloud services. │
│     "I'd use RDS for the database, S3 for file storage,        │
│      ElastiCache for the session store, and ECS for compute."   │
│     You can now say this with understanding, not memorization.  │
│                                                                 │
│  FINAL DELIVERABLE:                                             │
│  └─ Production-deployed SaaS backend.                           │
│     Docker Compose for local dev (you have this).               │
│     Cloud deployment for production (you can now do this).      │
│     README with architecture diagram — including cloud services.│
│                                                                 │
│  YOUR CAREER:                                                   │
│  └─ Every backend job posting mentions cloud.                   │
│     "Experience with AWS/GCP" isn't asking you to be a          │
│     DevOps engineer. It's asking: "Do you understand            │
│     where your code runs in production?"                        │
│     After today, you do.                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```