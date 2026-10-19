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