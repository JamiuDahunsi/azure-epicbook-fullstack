# Enterprise Full-Stack Deployment: The EpicBook on Microsoft Azure

[![Azure](https://img.shields.io/badge/Cloud-Microsoft_Azure-0078D4?logo=microsoft-azure&logoColor=white)](https://azure.microsoft.com/)
[![Ubuntu](https://img.shields.io/badge/OS-Ubuntu_24.04_LTS-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Node.js](https://img.shields.io/badge/Backend-Node.js_v20_LTS-339933?logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![Nginx](https://img.shields.io/badge/Web_Server-Nginx_Reverse_Proxy-009639?logo=nginx&logoColor=white)](https://nginx.org/)
[![MySQL](https://img.shields.io/badge/Database-Azure_MySQL_Flexible_Server-4479A1?logo=mysql&logoColor=white)](https://azure.microsoft.com/en-us/products/mysql/)
[![Architecture](https://img.shields.io/badge/Design-Multi--Tier_Private_VNet-blue)](#architecture-overview)

---

## 📌 Executive Summary

This project demonstrates the design, deployment, and security hardening of a production-grade full-stack e-commerce web application (**The EpicBook**) on **Microsoft Azure**.

The architecture decouples the public-facing application compute tier from the database tier. The application runtime (Node.js/Express) and web server (Nginx Reverse Proxy) run on an Ubuntu Virtual Machine inside a public subnet, while the managed relational database (**Azure Database for MySQL Flexible Server**) is strictly isolated inside a private subnet using **Virtual Network Integration** and dedicated subnet delegation. Public internet access to the database is disabled, enforcing the principle of least privilege and zero-trust data protection.

---

## 🏗️ Architecture Overview

The infrastructure separates compute and data layers across dedicated subnets within a custom Azure Virtual Network (`10.0.0.0/16`).

```mermaid
flowchart TD
    subgraph Internet ["Public Internet"]
        Client["Browser Client / Shopper"]
        Admin["DevOps / Admin Terminal (SSH)"]
    end

    subgraph Azure ["Microsoft Azure Cloud (Region: South Africa North)"]
        subgraph VNet ["Virtual Network: epicbook-vnet (10.0.0.0/16)"]
            
            subgraph PublicSubnet ["Public Application Subnet: public-subnet (10.0.1.0/24)"]
                NSG_Pub["Public NSG (epicbook-public-nsg)<br/>Inbound: Port 80 (HTTP) & Port 22 (SSH)"]
                PIP["Public IP: 40.123.244.184"]
                NIC["Network Interface: epicbook-vm-nic (10.0.1.4 / 10.0.1.5)"]
                
                subgraph VM ["Azure Virtual Machine: epicbook-vm"]
                    Nginx["Nginx Reverse Proxy<br/>(Listens: Port 80)"]
                    NodeApp["EpicBook Backend (Node.js/Express)<br/>(Listens: Internal Port 8080)"]
                    Sequelize["Sequelize ORM<br/>(TLS/SSL Transport)"]
                end
            end

            subgraph PrivateSubnet ["Private Database Subnet: private-db-subnet (10.0.2.0/24)"]
                NSG_Priv["Private NSG (epicbook-private-nsg)<br/>Inbound: Port 3306 Restricted to 10.0.1.0/24"]
                Delegation["Subnet Delegation:<br/>Microsoft.DBforMySQL/flexibleServers"]
                MySQL["Azure Database for MySQL Flexible Server<br/>epicbook-mysql-jamiu.mysql.database.azure.com<br/>(Private IP Only - No Public Access)"]
                DB[("Database: bookstore<br/>Tables: Author, Book, Cart, Checkout")]
            end

        end
    end

    Client -->|HTTP :80| PIP
    Admin -->|SSH :22| PIP
    PIP --> NIC
    NIC --> NSG_Pub
    NSG_Pub --> Nginx
    Nginx -->|Internal Proxy Pass| NodeApp
    NodeApp --> Sequelize
    Sequelize -->|TCP :3306 (SSL/TLS)| NSG_Priv
    NSG_Priv --> MySQL
    MySQL --> DB
```

---

## ⚙️ Core Technical Specifications

| Component | Specification | Purpose |
| :--- | :--- | :--- |
| **Cloud Provider** | Microsoft Azure | Enterprise cloud hosting and resource lifecycle management |
| **Region** | `South Africa North` | Low-latency deployment closest to user geography |
| **Virtual Network** | `epicbook-vnet` (`10.0.0.0/16`) | Isolated private network container |
| **Public Subnet** | `public-subnet` (`10.0.1.0/24`) | Houses internet-facing Virtual Machine and NIC |
| **Private Subnet** | `private-db-subnet` (`10.0.2.0/24`) | Houses isolated MySQL database with subnet delegation |
| **Subnet Delegation** | `Microsoft.DBforMySQL/flexibleServers` | Grants Azure resource management rights for dynamic routing |
| **Compute VM** | Ubuntu 24.04 LTS (`Standard_B2ats_v2`) | Application host running Nginx and Node.js backend |
| **Web Server** | Nginx (`v1.24.0`) | Reverse proxy forwarding HTTP Port 80 traffic to internal Port 8080 |
| **Backend Runtime** | Node.js (`v20.20.2 LTS`) & npm (`10.8.2`) | Executes Express application server and business logic |
| **Database Engine** | Azure MySQL Flexible Server (`v8.4`) | Fully managed relational database engine with automated backups |
| **Network Security** | Network Security Groups (NSGs) | Stateful firewalls controlling ingress and egress traffic per subnet |

---

## 🔒 Security & Networking Design

### 1. Network Segmentation & Database Isolation
* **Zero Public Endpoint:** The MySQL database has **no public IP address**. It exists solely within `private-db-subnet` (`10.0.2.0/24`). Port scanning or brute-force attacks from the internet cannot reach port 3306.
* **Private DNS Zone:** Internal resolution (`epicbook-mysql-jamiu.private.mysql.database.azure.com`) resolves directly to internal private IP space.

### 2. Micro-Segmentation via Network Security Groups (NSGs)

#### Public Subnet NSG (`epicbook-public-nsg`):
| Priority | Rule Name | Port | Protocol | Source | Action | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `100` | `Allow_HTTP` | `80` | TCP | `Any` | **Allow** | Permits public internet visitors to access web store |
| `110` | `Allow_SSH` | `22` | TCP | `Any` / Admin IP | **Allow** | Encrypted command-line administration |
| `65500` | `DenyAllInbound` | `*` | Any | `Any` | **Deny** | Default drop for all other unsolicited ingress |

#### Private Subnet NSG (`epicbook-private-nsg`):
| Priority | Rule Name | Port | Protocol | Source | Action | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `100` | `Allow-MySQL-From-Public-Subnet` | `3306` | TCP | **`10.0.1.0/24`** | **Allow** | **Restricts MySQL access strictly to the VM subnet** |
| `65500` | `DenyAllInbound` | `*` | Any | `Any` | **Deny** | Blocks all direct outside access to port 3306 |

### 3. Encrypted Transit (`require_secure_transport = ON`)
Azure Database for MySQL Flexible Server enforces TLS/SSL encryption for all socket handshakes. The application is configured using Sequelize `dialectOptions`:
```json
"dialectOptions": {
  "ssl": {
    "rejectUnauthorized": false
  }
}
```
This guarantees that all database queries and customer records (e.g., checkout and cart data) are encrypted in flight across the virtual network.

---

## 🚀 Step-by-Step Implementation

### Step 1: Network & Security Group Provisioning
1. Provisioned Resource Group `epicbook-rg` in `South Africa North`.
2. Created Virtual Network `epicbook-vnet` (`10.0.0.0/16`).
3. Partitioned subnets:
   * `public-subnet`: `10.0.1.0/24`
   * `private-db-subnet`: `10.0.2.0/24` with delegation to `Microsoft.DBforMySQL/flexibleServers`.
4. Created `epicbook-public-nsg` and associated it with `public-subnet`.
5. Created `epicbook-private-nsg` with restricted rule for port 3306 and associated it with `private-db-subnet`.
6. Created Network Interface `epicbook-vm-nic` with attached Public IP (`40.123.244.184`).

### Step 2: Virtual Machine & Runtime Setup
1. Deployed Ubuntu Linux VM (`epicbook-vm`) attached to `epicbook-vm-nic`.
2. Installed essential utilities and runtimes:
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install -y nginx git mysql-client curl
   curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
   sudo apt install -y nodejs
   ```
3. Verified installed toolchain:
   ```bash
   node -v && npm -v && git --version && nginx -v && mysql --version
   ```

### Step 3: Database Initialization & Seeding
1. Provisioned **Azure Database for MySQL Flexible Server** with Private Access (VNet Integration) in `private-db-subnet`.
2. Cloned application repository to VM:
   ```bash
   git clone https://github.com/pravinmishraaws/theepicbook.git ~/theepicbook
   ```
3. Connected from VM over private network and initialized database:
   ```bash
   mysql -h epicbook-mysql-jamiu.mysql.database.azure.com -u azureuser -p
   CREATE DATABASE bookstore;
   EXIT;
   ```
4. Streamed schema definition and seed inventory into MySQL:
   ```bash
   # Import Schema (Creates Author, Book, Cart, Checkout tables)
   mysql -h <HOST> -u azureuser -p bookstore < ~/theepicbook/db/BuyTheBook_Schema.sql

   # Seed Author Records (53 authors)
   mysql -h <HOST> -u azureuser -p bookstore < ~/theepicbook/db/author_seed.sql

   # Seed Book Catalog (54 books with relational foreign keys)
   mysql -h <HOST> -u azureuser -p bookstore < ~/theepicbook/db/books_seed.sql
   ```

### Step 4: Reverse Proxy & Application Deployment
1. Configured Nginx as reverse proxy (`/etc/nginx/sites-available/epicbook`):
   ```nginx
   server {
       listen 80;
       server_name _;

       location / {
           proxy_pass http://127.0.0.1:8080;
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```
2. Activated configuration and reloaded Nginx:
   ```bash
   sudo rm -f /etc/nginx/sites-enabled/default
   sudo ln -s /etc/nginx/sites-available/epicbook /etc/nginx/sites-enabled/epicbook
   sudo nginx -t && sudo systemctl restart nginx
   ```
3. Configured database credentials in `config/config.json` with SSL transport options.
4. Installed Node.js dependencies and launched backend:
   ```bash
   cd ~/theepicbook
   npm install
   npm run start
   ```

---

## 🧪 Verification & End-to-End Testing

### 1. Reverse Proxy & Service Health Check
```bash
$ curl -I http://localhost:8080
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 24858
```

### 2. Active Socket Verification
```bash
$ sudo ss -tulpn | grep 8080
tcp   LISTEN 0      511        *:8080        *:*    users:(("node",pid=4343,fd=21))
```

### 3. Dynamic Database Persistence Test
When an item was added to the cart via the public web interface (`http://40.123.244.184`), the action was queried directly from the Azure MySQL instance:

```sql
mysql> SELECT * FROM Cart;
+----+----------+-------+---------------------+---------------------+
| id | quantity | price | createdAt           | updatedAt           |
+----+----------+-------+---------------------+---------------------+
|  1 |        1 | 17.00 | 2026-10-28 07:35:16 | 2026-10-28 07:35:16 |
+----+----------+-------+---------------------+---------------------+
1 row in set (0.01 sec)
```

---

## 📷 Deployment Proof & Evidence Gallery

### Cloud Infrastructure & Networking
| Deliverable | Description | Evidence Output |
| :--- | :--- | :--- |
| **Virtual Network & Subnets** | Address space `10.0.0.0/16`, `public-subnet` (`10.0.1.0/24`), and `private-db-subnet` (`10.0.2.0/24`) | Subnet blade showing MySQL delegation |
| **Public & Private NSGs** | Port 80 & 22 open on public subnet; Port 3306 restricted to `10.0.1.0/24` on private subnet | Inbound security rules table |
| **Network Interface (NIC)** | `epicbook-vm-nic` attached to `public-subnet` with public IP association | IP Configurations blade |
| **Ubuntu Compute Instance** | `epicbook-vm` running Ubuntu Linux in `South Africa North` | Virtual Machine Overview |
| **Managed MySQL Instance** | `epicbook-mysql-jamiu` status Ready with Private Access | Azure Database for MySQL Overview |
| **Subnet Delegation** | MySQL server attached to `private-db-subnet` | MySQL Networking blade |

### Application & Runtime Verification
| Deliverable | Description | Command / Output |
| :--- | :--- | :--- |
| **Installed Toolchain** | Node.js v20, npm 10, Git 2.43, Nginx 1.24, MySQL Client 8.0 | `node -v && npm -v && git --version && nginx -v && mysql --version` |
| **Relational Database Seeds** | 53 Author records and 54 Book records verified in `bookstore` | `SHOW TABLES; SELECT * FROM Author; SELECT * FROM Book;` |
| **Application Dependencies** | Dependencies compiled into `node_modules/` | `npm install && ls -l` |
| **Reverse Proxy Configuration** | Nginx configured to pass traffic from Port 80 to Port 8080 | `cat /etc/nginx/sites-available/epicbook && sudo nginx -t` |
| **Active Backend Listener** | Node process actively listening on port 8080 | `sudo ss -tulpn \| grep 8080` |
| **Live Web Application** | Production bookstore running on `http://40.123.244.184` | Browser view showing live catalog and cart controls |
| **Database-Backed Transaction** | E-commerce action verified in database | `SELECT * FROM Cart;` showing item added via web UI |

---

## 💡 Key Learnings & Engineering Takeaways

1. **Separation of Concerns:** Running the application logic and the database on separate tiers eliminates single points of failure and prevents database starvation if web traffic surges.
2. **Defense in Depth:** Even within the same cloud datacenter, placing databases in non-routable private subnets and locking NSG ingress to internal CIDR blocks satisfies PCI-DSS and financial regulatory requirements.
3. **Reverse Proxy Architecture:** Exposing backend application runtimes directly to the public internet is insecure. Running Nginx in front of Node.js shields the application process, handles HTTP connection pooling, and simplifies future SSL certificate management.
4. **Resilient Data Persistence:** Decoupling compute from storage ensures that even if the virtual machine is destroyed, resized, or updated, all business records remain secure in the managed database service.

---

## 👤 Author

* **Name:** Jamiu Dahunsi
* **Role:** Network & Cloud Infrastructure Engineer
* **Organization:** Zone Payment Network
* **Project:** Full-Stack Cloud Deployment on Microsoft Azure
