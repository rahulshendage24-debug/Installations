# Splunk 2-Node EC2 Setup on Amazon Linux (AL2 / AL2023)

On Amazon Linux, you use `.rpm` packages (via `rpm` or `dnf`/`yum`) instead of Debian `.deb` packages. In addition, Amazon Linux logs system authentication to `/var/log/secure` instead of `/var/log/auth.log`.

---

## Architecture & Machine Roles

| Node | Role | Splunk Component | Package Format | Log File | Inbound Security Group Ports |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **EC2 Instance 1** | **Master** / Server | Splunk Enterprise | `.rpm` | N/A | `22` (SSH), `8000` (Web UI), `9997` (Forwarder Ingestion) |
| **EC2 Instance 2** | **Worker** / Client | Splunk Universal Forwarder | `.rpm` | `/var/log/secure` | `22` (SSH) |

---

## Part 1: EC2 Instance 1 — The Master (Splunk Server)

SSH into **EC2 Instance 1** (`ec2-user@<EC2_1_PUBLIC_IP>`):

### 1. Download & Install Splunk Enterprise (.rpm)
```bash
# Update package repositories and ensure wget is installed
sudo dnf update -y || sudo yum update -y
sudo dnf install -y wget || sudo yum install -y wget

# Download the Splunk Enterprise RPM package
wget -O splunk-enterprise.rpm "[https://download.splunk.com/products/splunk/releases/9.4.0/linux/splunk-9.4.0-6b4ebe426ca6.x86_64.rpm](https://download.splunk.com/products/splunk/releases/9.4.0/linux/splunk-9.4.0-6b4ebe426ca6.x86_64.rpm)"

# Install using RPM
sudo rpm -ivh splunk-enterprise.rpm

### 2. Initialize Service & Configure Admin Credentials

```bash
# Accept license non-interactively and seed your admin password
sudo /opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt --seed-passwd MyAdminPass123!

# Configure systemd boot-start under the splunk system user
sudo /opt/splunk/bin/splunk enable boot-start -user splunk

### 3. Open Ingestion Port (9997) & Create Destination Index

```bash
# Open receiver listening port on TCP 9997
sudo -u splunk /opt/splunk/bin/splunk enable listen 9997 -auth admin:MyAdminPass123!

# Create an index named 'student_logs'
sudo -u splunk /opt/splunk/bin/splunk add index student_logs -auth admin:MyAdminPass123!

# Verify that port 9997 is actively listening
sudo -u splunk /opt/splunk/bin/splunk display listen -auth admin:MyAdminPass123!

### 4. Verification

Open your browser and navigate to:

```text
http://<EC2_1_PUBLIC_IP>:8000
Log in using:

Username: admin
Password: MyAdminPass123!

## Part 2: EC2 Instance 2 — The Worker / Client (Universal Forwarder)

SSH into **EC2 Instance 2** (`ec2-user@<EC2_2_PUBLIC_IP>`):

### 1. Download & Install Universal Forwarder (.rpm)

```bash
# Update package repositories and ensure wget is installed
sudo dnf update -y || sudo yum update -y
sudo dnf install -y wget || sudo yum install -y wget

# Download the Universal Forwarder RPM package
wget -O splunkforwarder.rpm "[https://download.splunk.com/products/universalforwarder/releases/9.4.0/linux/splunkforwarder-9.4.0-6b4ebe426ca6.x86_64.rpm](https://download.splunk.com/products/universalforwarder/releases/9.4.0/linux/splunkforwarder-9.4.0-6b4ebe426ca6.x86_64.rpm)"

# Install using RPM
sudo rpm -ivh splunkforwarder.rpm

### 2. Initialize Forwarder & Set Local Credentials

```bash
# Accept license and seed local forwarder admin credentials
sudo /opt/splunkforwarder/bin/splunk start --accept-license --answer-yes --no-prompt --seed-passwd ForwarderPass123!

# Enable auto-start on boot
sudo /opt/splunkforwarder/bin/splunk enable boot-start

### 3. Connect Worker to EC2 1 (Master)

> **Note:** Replace `<EC2_1_PRIVATE_IP>` with the Private IPv4 of EC2 1 (if both instances are in the same VPC), or use its Public IP.

```bash
# Point the forwarder to EC2 1 on ingestion port 9997
sudo /opt/splunkforwarder/bin/splunk add forward-server <EC2_1_PRIVATE_IP>:9997 -auth admin:ForwarderPass123!

### 4. Configure Log Monitoring for Amazon Linux

> **Important:** On Amazon Linux (RHEL family), authentication events are logged to `/var/log/secure` (not `/var/log/auth.log`).

```bash
# Grant read access to the secure log
sudo chmod 644 /var/log/secure

# Monitor /var/log/secure and forward to the 'student_logs' index
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/secure -index student_logs -sourcetype linux_secure -auth admin:ForwarderPass123!

### 5. Verification on Worker

Run the following commands on **EC2 Instance 2** to confirm that the connection and log monitoring are configured properly:

#### A. Check Forwarding Server Connection
```bash
# Verify forward-server status
sudo /opt/splunkforwarder/bin/splunk list forward-server -auth admin:ForwarderPass123!

## Part 3: Live Classroom Demonstration

### 1. Prepare Splunk Web (EC2 1)

1. Open your browser and navigate to:
   ```text
   http://<EC2_1_PUBLIC_IP>:8000

   Navigate to Search & Reporting.

Type the following query into the search bar:

1.Splunk SPL
2. index=student_logs
3. Set the time filter to Real-time (30 second window) or Last 15 minutes.
4. Click Search (magnifying glass icon).

### 3. Live Demonstration with Students (EC2 1)

Show students the real-time pipeline in action:

#### 1. Raw Log Ingestion
Point out the new lines arriving in the search results with:
- `source=/var/log/secure`
- `sourcetype=linux_secure`
- `host=<Worker_EC2_Hostname>`

---

#### 2. Filter by Attack
Demonstrate keyword filtering to isolate failed login attempts:

```spl
index=student_logs "Failed password"

### 3. Aggregate & Extract Targets

Show how the pipe (`|`) operator transforms raw text into structured operational analytics by identifying targeted accounts and source IPs:

```spl
index=student_logs "Failed password"
| stats count by user, src_ip
| sort - count

### 4. Instant Dashboard

1. Switch from the **Statistics** tab to the **Visualization** tab.
2. Select **Bar Chart**, **Column Chart**, or **Pie Chart** from the chart dropdown.
3. Click **Save As** > **Dashboard Panel** to show students how quickly ad-hoc threat investigations turn into executive security dashboards.
