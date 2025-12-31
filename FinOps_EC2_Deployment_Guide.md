# FinOps & Compliance Canvas - EC2 Deployment Guide
## Amazon Linux 2023 Deployment

---

## Application Overview

This is a multi-module Streamlit application with:
- Azure AD SSO Authentication
- Firebase User Management
- AWS Multi-Account Integration
- Claude AI Integration
- SSL Certificates (finopsncompliance.com)

---

## Step 1: Launch EC2 Instance

### Recommended Instance Configuration

| Setting | Value |
|---------|-------|
| **AMI** | Amazon Linux 2023 |
| **Instance Type** | `t3.large` (2 vCPU, 8GB RAM) - recommended for this app |
| **Storage** | 30 GB gp3 |
| **Key Pair** | Your existing or new key |

### Security Group Rules

| Type | Port | Source | Purpose |
|------|------|--------|---------|
| SSH | 22 | Your IP | Remote access |
| HTTP | 80 | 0.0.0.0/0 | Web traffic |
| HTTPS | 443 | 0.0.0.0/0 | SSL traffic |
| Custom TCP | 8501 | 0.0.0.0/0 | Streamlit (optional) |

---

## Step 2: Connect and Initial Setup

```bash
# Connect to EC2
ssh -i "your-key.pem" ec2-user@<your-ec2-ip>

# Switch to root
sudo su

# Update system
dnf update -y
```

---

## Step 3: Install Dependencies

```bash
# Install Python 3.11
dnf install python3.11 python3.11-pip python3.11-devel -y

# Install system dependencies
dnf install -y gcc gcc-c++ make git nginx

# Install additional libraries
dnf install -y freetype-devel libpng-devel libjpeg-turbo-devel

# Install dos2unix for file conversion
dnf install -y dos2unix
```

---

## Step 4: Create Application Directory

```bash
# Create directory
mkdir -p /opt/finops-app
chown ec2-user:ec2-user /opt/finops-app

# Navigate to directory
cd /opt/finops-app

# Create virtual environment
python3.11 -m venv venv
source venv/bin/activate

# Upgrade pip
pip install --upgrade pip setuptools wheel
```

---

## Step 5: Upload Application Files via FileZilla

Connect FileZilla to your EC2 and upload ALL files to `/opt/finops-app/`:

```
/opt/finops-app/
├── streamlit_app.py
├── requirements.txt
├── account_lifecycle_enhanced.py
├── ai_configuration_assistant_complete.py
├── ai_threat_scene_6_PRODUCTION.py
├── auth_azure_sso.py
├── auth_database_firebase.py
├── aws_connector.py
├── aws_finops_data.py
├── azure_sso_auth.py
├── batch_remediation_production.py
├── claude_predictions.py
├── code_generation_production.py
├── crewai_finops_agents.py
├── eks_container_vulnerability_module.py
├── eks_remediation_complete.py
├── eks_vulnerability_enterprise_complete.py
├── enterprise_module.py
├── finops_live_data.py
├── finops_module_enhanced_complete.py
├── finops_scene_7_complete.py
├── linux_distribution_remediation_MERGED_ENHANCED.py
├── multi_account_connector.py
├── multi_account_policy_manager.py
├── pipeline_simulator.py
├── policy_as_code_platform.py
├── scp_policy_engine.py
├── scp_scene_5_enhanced.py
├── tech_guardrails_enterprise.py
├── unified_remediation_dashboard.py
├── windows_server_remediation_MERGED_ENHANCED.py
├── certs/
│   ├── finopsncompliance.com-chain.pem
│   ├── finopsncompliance.com-crt.pem
│   └── finopsncompliance.com-key.pem
└── .streamlit/
    └── secrets.toml
```

**Important**: Do NOT upload the `__pycache__/` or `.git/` folders.

---

## Step 6: Install Python Dependencies

```bash
cd /opt/finops-app
source venv/bin/activate

# Install requirements
pip install -r requirements.txt
```

---

## Step 7: Create Streamlit Configuration

```bash
mkdir -p /opt/finops-app/.streamlit
```

Create config file:

```bash
cat > /opt/finops-app/.streamlit/config.toml << 'EOF'
[server]
port = 8501
address = "0.0.0.0"
headless = true
enableCORS = false
enableXsrfProtection = true
maxUploadSize = 200

[browser]
gatherUsageStats = false

[theme]
primaryColor = "#1e40af"
backgroundColor = "#ffffff"
secondaryBackgroundColor = "#f0f2f6"
textColor = "#262730"
EOF
```

---

## Step 8: Fix Line Endings (Important!)

After uploading via FileZilla from Windows:

```bash
cd /opt/finops-app

# Fix all Python files
dos2unix *.py

# Fix secrets.toml
dos2unix .streamlit/secrets.toml

# Fix requirements.txt
dos2unix requirements.txt
```

---

## Step 9: Create Systemd Service

```bash
cat > /etc/systemd/system/finops-app.service << 'EOF'
[Unit]
Description=FinOps Compliance Canvas Streamlit Application
After=network.target

[Service]
Type=simple
User=ec2-user
Group=ec2-user
WorkingDirectory=/opt/finops-app
ExecStart=/opt/finops-app/venv/bin/streamlit run streamlit_app.py --server.port 8501 --server.address 0.0.0.0
Restart=always
RestartSec=10
StandardOutput=append:/var/log/finops-app.log
StandardError=append:/var/log/finops-app-error.log

[Install]
WantedBy=multi-user.target
EOF

# Create log files
touch /var/log/finops-app.log
touch /var/log/finops-app-error.log
chown ec2-user:ec2-user /var/log/finops-app*.log

# Reload systemd
systemctl daemon-reload
```

---

## Step 10: Configure Nginx with SSL

### Option A: Using Your Existing SSL Certificates

Your app includes SSL certificates for `finopsncompliance.com`. If your domain points to this EC2:

```bash
# Copy certificates to nginx directory
mkdir -p /etc/nginx/ssl
cp /opt/finops-app/certs/finopsncompliance.com-crt.pem /etc/nginx/ssl/
cp /opt/finops-app/certs/finopsncompliance.com-key.pem /etc/nginx/ssl/
cp /opt/finops-app/certs/finopsncompliance.com-chain.pem /etc/nginx/ssl/

# Set permissions
chmod 600 /etc/nginx/ssl/*.pem
```

Create Nginx config with SSL:

```bash
cat > /etc/nginx/conf.d/finops.conf << 'EOF'
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name finopsncompliance.com www.finopsncompliance.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS Server
server {
    listen 443 ssl;
    server_name finopsncompliance.com www.finopsncompliance.com;

    ssl_certificate /etc/nginx/ssl/finopsncompliance.com-chain.pem;
    ssl_certificate_key /etc/nginx/ssl/finopsncompliance.com-key.pem;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://127.0.0.1:8501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
        proxy_buffering off;
    }

    location /_stcore/stream {
        proxy_pass http://127.0.0.1:8501/_stcore/stream;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }
}
EOF
```

### Option B: HTTP Only (No Domain)

If you don't have a domain yet:

```bash
cat > /etc/nginx/conf.d/finops.conf << 'EOF'
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400;
        proxy_buffering off;
    }

    location /_stcore/stream {
        proxy_pass http://127.0.0.1:8501/_stcore/stream;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }
}
EOF
```

---

## Step 11: Start Services

```bash
# Test Nginx configuration
nginx -t

# Enable and start Nginx
systemctl enable nginx
systemctl start nginx

# Enable and start Streamlit app
systemctl enable finops-app
systemctl start finops-app

# Check status
systemctl status finops-app
systemctl status nginx
```

---

## Step 12: Update Azure AD Redirect URI

If using Azure AD SSO, update the redirect URI in Azure Portal:

1. Go to **Azure Portal** → **Azure Active Directory** → **App registrations**
2. Select your app
3. Go to **Authentication**
4. Update **Redirect URI** to:
   - `https://finopsncompliance.com` (if using domain)
   - `http://<your-ec2-ip>:8501` (if using IP)

Also update in `/opt/finops-app/.streamlit/secrets.toml`:

```toml
[azure_ad]
redirect_uri = "https://finopsncompliance.com"  # or your EC2 IP
```

---

## Step 13: Access Your Application

- **With Domain + SSL**: `https://finopsncompliance.com`
- **With IP (HTTP)**: `http://<your-ec2-ip>`
- **Direct Streamlit**: `http://<your-ec2-ip>:8501`

---

## Useful Commands

```bash
# View logs
tail -f /var/log/finops-app.log
tail -f /var/log/finops-app-error.log

# Restart application
systemctl restart finops-app

# Restart Nginx
systemctl restart nginx

# Check status
systemctl status finops-app

# View live journal logs
journalctl -u finops-app -f
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Module not found | Check all .py files uploaded |
| Line ending errors | Run `dos2unix` on all files |
| Permission denied | Check file ownership: `chown -R ec2-user:ec2-user /opt/finops-app` |
| SSL errors | Verify certificate paths and permissions |
| Azure SSO fails | Update redirect URI in Azure Portal |
| Firebase errors | Verify secrets.toml has correct credentials |

---

## Security Reminders

1. Your `secrets.toml` contains sensitive API keys - ensure it's not publicly accessible
2. Restrict SSH access to your IP only
3. Consider using AWS Secrets Manager for production
4. Regularly rotate API keys and credentials

---

*FinOps & Compliance Canvas EC2 Deployment Guide*
