# GCP Private Flask App with Regional Load Balancer

## Overview
This project demonstrates an end-to-end deployment of a **private Flask application** on GCP, exposed via a Regional HTTP(S) Load Balancer. All infra was created manually via the GCP Console. Inspired by Abhishek’s workflow, but designed and implemented independently.

***

## Architecture Diagram

![Architecture diagram for GCP Private Flask App with Regional Load Balancer](https://user-gen-media-assets.s3.amazonaws.com/gemini_images/eb8f2a04-17cb-4ae7-a7da- Key Technologies
- **Google Cloud Platform:** VPC, Compute Engine, MIGs, Cloud NAT, LB
- **Python:** Flask web app, Gunicorn WSGI server
- **Startup Script:** Full app install & launch automation
- **GoDaddy + Cloud DNS:** Planned custom domain

***

## Steps Implemented

1. **Custom VPC Setup**
   - Created `prod-vpc` with three subnets (public, app, proxy-only) in `asia-south1`.
2. **Cloud NAT & Router**
   - NAT for secure egress from private instances.
3. **Firewall Rules**
   - Health-check and load balancer data-plane rules on port 8080.
4. **Instance Template & Startup Script**
   - Used a bash script for unattended Flask+Gunicorn deployment.
5. **Managed Instance Group**
   - Configured autoscaling group in private subnet, no external IPs.
6. **Regional Load Balancer**
   - HTTP traffic on port 80, backend group on port 8080 connected to MIG.
7. **(Optional) Custom Domain**
   - Documented GoDaddy-to-GCP DNS steps for domain mapping.

***

## Startup Script

```bash
#!/bin/bash
set -e

# Update system and install Python3 and venv
apt-get update -y
apt-get install -y python3-pip python3-venv

# Create 'appuser' if not exist
id -u appuser &>/dev/null || useradd -m -s /bin/bash appuser

sudo -u appuser bash <<'EOF'
cd ~
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install flask gunicorn

cat > app.py <<PY
from flask import Flask
app = Flask(__name__)

@app.route("/")
def index():
    return "Hello from Flask on GCP (private subnet)!"

@app.route("/health")
def health():
    return "ok", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
PY
EOF

cat >/etc/systemd/system/flask.service <<'UNIT'
[Unit]
Description=Flask via Gunicorn in venv
After=network.target

[Service]
User=appuser
WorkingDirectory=/home/appuser
ExecStart=/home/appuser/venv/bin/gunicorn -w 2 -b 0.0.0.0:8080 app:app
Restart=always

[Install]
WantedBy=multi-user.target
UNIT

systemctl daemon-reload
systemctl enable --now flask.service

---

## Validation

The deployment can be verified by:
- **Browser:**  
  Visit `http://<Load_Balancer_IP>/` — should display:  
  `"Hello from Flask on GCP (private subnet)!"`

- **Health Endpoint:**  
  Visit `http://<Load_Balancer_IP>/health` or run:  

curl http://<Load_Balancer_IP>/health

Expected output:  
`ok`

---

## Why This Project?

- **Cloud/DevOps Practice:**  
Real GCP networking, compute, firewall, and load balancer setup—matching industry expectations for DevOps/cloud roles.
- **Automation:**  
Hands-on with startup scripting and instance group orchestration: no manual remote server setup.
- **Showcasing Skills:**  
Well-documented architecture and steps for use in interviews or portfolio.
- **Future-ready:**  
Architecture planned for easy extension to CI/CD and domain/SSL integration.

---
