# Installing Trivy on Amazon Linux (AL2 & AL2023)

Step-by-step instructions for installing Aqua Security's Trivy vulnerability scanner on Amazon Linux via the official repository.

---

### 1. Add the Repository Configuration

Create the Trivy repository file:

```bash
sudo tee /etc/yum.repos.d/trivy.repo << 'EOF'
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://aquasecurity.github.io/trivy-repo/rpm/public.key
EOF
```

---

### 2. Update Metadata & Install Trivy


Run the appropriate command depending on your Amazon Linux version:

#### For Amazon Linux 2023:
```bash
sudo dnf update -y
sudo dnf install -y trivy
```

#### For Amazon Linux 2:
```bash
sudo yum update -y
sudo yum install -y trivy
```

---

### 3. Verify Installation

Check that Trivy was successfully installed:

```bash
trivy --version
```

# Trivy Commands Cheat Sheet

A quick reference guide for essential Aqua Security Trivy commands.

---

### 1. Scan a Container Image
Scan a local or remote container image for vulnerabilities:

```bash
trivy image imagename
```

---

### 2. Scan Filesystem / Directory
Scan application dependencies and configuration files in a specified folder:

```bash
trivy fs --scanners vuln,config Folder_name_OR_Path
```

> **Note:** `--scanners` is the modern flag replacing the legacy `--security-checks` option in newer versions of Trivy.

---

### 3. Filter by Severity Level
Focus on priority vulnerabilities by filtering for `HIGH` and `CRITICAL` findings:

```bash
trivy image --severity HIGH,CRITICAL image_name
```

---

### 4. Output Results to JSON File
Save the scan report in JSON format for parsing, archiving, or CI/CD pipelines:

```bash
trivy image -f json -o results.json image_name
```

---

### 5. Scan a Remote Repository
Audit a remote Git repository without needing to clone it locally first:

```bash
trivy repo repo-url
```

---

### 6. Scan a Kubernetes Cluster
Audit workload misconfigurations and vulnerabilities across a running cluster:

```bash
trivy k8s --report summary cluster
```
