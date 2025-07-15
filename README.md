#  Clustered Apache Guacamole Deployment with Docker Compose

This project demonstrates how to deploy a **high-availability, load-balanced remote desktop gateway** using Apache Guacamole and Docker Compose. It includes PostgreSQL for backend storage, Nginx for load balancing, and two Guacamole frontends to ensure scalability and redundancy.

---

##  What is Apache Guacamole?

Apache Guacamole is a **clientless remote desktop gateway** that supports standard protocols like:

- **RDP** (Remote Desktop Protocol)
- **VNC**
- **SSH**

Users access remote desktops through a web browser — **no need to install any client software**.

---

##  Project Structure

```text
guacamole-project/
├── docker-compose.yml              # Main Docker Compose file
├── nginx/
│   └── default.conf                # Nginx load balancing config
├── guacamole/
│   └── guacamole.properties        # Guacamole config file (optional)
├── init/
│   └── initdb.sql                  # SQL schema to initialize PostgreSQL
└── README.md
```

---

##  What is Docker Compose?

**Docker Compose** is a tool that lets you define and manage **multi-container Docker applications** using a single YAML file.

Instead of running containers manually, Compose lets you describe:

- What containers to run
- How they connect
- What environment variables they need
- What volumes or ports they expose

You run everything with just:

```bash
docker-compose up -d
```

---

##  Services in docker-compose.yml

### 1. **postgres**
- This is the **database container**.
- Stores all Guacamole user accounts, configurations, and connection settings.
- Uses the `init/initdb.sql` file to create the necessary database tables on first startup.

```yaml
postgres:
  image: postgres
  environment:
    POSTGRES_DB: guacamole_db
    POSTGRES_USER: guac_admin
    POSTGRES_PASSWORD: CompNet2486
  volumes:
    - ./init/initdb.sql:/docker-entrypoint-initdb.d/initdb.sql
```

---

### 2. **guacd**
- Guacamole’s **daemon** that actually handles the RDP, VNC, and SSH connections.
- All frontend Guacamole containers send remote session instructions to this daemon.

```yaml
guacd:
  image: guacamole/guacd
```

---

### 3. **guacamole1 & guacamole2**
- These are the **frontend web interfaces**.
- Users access these via Nginx (load balancer).
- Each one connects to `guacd` and `postgres` to function.

```yaml
guacamole1:
  image: guacamole/guacamole
  environment:
    GUACD_HOSTNAME: guacd
    POSTGRES_HOST: postgres
    POSTGRES_DATABASE: guacamole_db
    POSTGRES_USER: guac_admin
    POSTGRES_PASSWORD: CompNet2486
  depends_on:
    - guacd
    - postgres

guacamole2:
  image: guacamole/guacamole
  environment: *same as above*
  depends_on:
    - guacd
    - postgres
```

---

### 4. **nginx**
- The **reverse proxy** and **load balancer**.
- Forwards traffic to `guacamole1` or `guacamole2`.
- Uses `nginx/default.conf` for configuration.

```yaml
nginx:
  image: nginx
  ports:
    - "80:80"
  volumes:
    - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
  depends_on:
    - guacamole1
    - guacamole2
```

---

##  Nginx Load Balancing

Inside `nginx/default.conf`:

```nginx
upstream guacamole_cluster {
    server guacamole1:8080;
    server guacamole2:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://guacamole_cluster;
        ...
    }
}
```

This allows round-robin load balancing between the two Guacamole web UIs.

---

##  guacamole.properties (Optional)

Used inside each Guacamole container to specify how to connect to services:

```properties
guacd-hostname: guacd
guacd-port: 4822
postgres-hostname: postgres
postgres-port: 5432
postgres-database: guacamole_db
postgres-username: guac_admin
postgres-password: CompNet2486
```

Alternatively, the same info can be passed using environment variables in `docker-compose.yml`.

---

##  How to Deploy (Step-by-Step)

### 1. Clone the Repository

```bash
git clone https://github.com/kiarashtmu/guacamole-project.git
cd guacamole-project
```

### 2. Generate `initdb.sql` (If not already done)

```bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgres > init/initdb.sql
```

### 3. Start the Environment

```bash
docker-compose up -d
```

### 4. Access the Interface

Go to:

 `http://localhost/guacamole`  
(use incognito to test multiple sessions)

Login with:

- **Username:** `guacadmin`
- **Password:** `guacadmin`

---

##  Load Balancing Test

1. Open multiple incognito tabs
2. Refresh the Guacamole login page
3. Run this to watch logs:

```bash
docker logs -f guacamole1
docker logs -f guacamole2
```

You'll see alternating requests, proving Nginx is balancing traffic.

---

##  Troubleshooting

| Issue                      | Fix |
|---------------------------|-----|
| Nginx returns 502 error   | Ensure guacamole1/2 are running and healthy |
| Database connection fails | Verify initdb.sql is mounted and Postgres is up |
| Can’t log in              | Try using `localhost:8080` directly |
| Can’t reach `/guacamole` | Check that Nginx is forwarding properly |

---

##  Push Changes to GitHub

```bash
git add .
git commit -m "Added detailed Compose explanation"
git push origin main
```

---


