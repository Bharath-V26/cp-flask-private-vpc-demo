# GCP Private Flask App with Regional Load Balancer

## Overview
This project demonstrates an end-to-end deployment of a **secure, private Python Flask application** on Google Cloud Platform (GCP), exposed via a **Regional External HTTP(S) Load Balancer**. The infrastructure includes a custom VPC network with multiple subnets, Cloud NAT for internet egress of private instances, firewall rules for security, a managed instance group running the Flask app via a startup script, and a planned custom domain setup with GoDaddy and Google Cloud DNS.

All infrastructure was created manually via the GCP Console for clarity and reproducibility, inspired by Abhishek's tutorial but fully designed and implemented independently.

---

## Architecture Diagram

![Architecture diagram for GCP Private Flask App with Regional Load Balancer](https://user-gen-media-assets.s3.amazonaws.com/gemini_images/eb8f2a04-17cb-4ae7-a7da-6feaf5cec447.png)

---

## Key Technologies

- **Google Cloud Platform:** VPC, Compute Engine, Managed Instance Groups (MIGs), Cloud NAT, Regional External HTTP(S) Load Balancer  
- **Python Flask:** Lightweight web framework for the app  
- **Gunicorn:** WSGI HTTP server to serve Flask app  
- **Startup Script:** Automates installation and app launch on VMs  
- **GoDaddy + Cloud DNS:** Planned custom domain mapping (documented but not live)  

---

## Steps Implemented

1. **Create Custom VPC and Subnets**  
   - Created a VPC named `prod-vpc` with three subnets:  
     - `subnet-public` (10.10.10.0/24)  
     - `subnet-app` (10.10.20.0/24) — where backend app instances reside  
     - `subnet-proxy-only` (10.10.30.0/24) — dedicated to load balancer proxy traffic  

2. **Set Up Cloud Router and Cloud NAT**  
   - Deployed Cloud Router (`router-asia-south1`) for dynamic routing in the region.  
   - Created Cloud NAT gateway (`nat-asia-south1`) to allow private VMs internet access without external IPs.  

3. **Firewall Rules Configuration**  
   - Allowed Google health check IP ranges to reach backend instances on TCP 8080 for load balancer health checks.  
   - Allowed load balancer data-plane traffic from `subnet-proxy-only` subnet on TCP 8080 only, enhancing security posture.  

4. **Instance Template & Startup Script**  
   - Created instance template `tmpl-flask-app`, with no external IP assigned.  
   - Startup script provisions Python3, pip, virtual environment, installs Flask and Gunicorn, creates the app, and configures a systemd service to run Gunicorn on port 8080.  

5. **Managed Instance Group (MIG)**  
   - Set up MIG `mig-flask-app` in a single zone `asia-south1-a`.  
   - Autoscaling configured between 2 and 4 backend instances in private subnet.  

6. **Regional External HTTP(S) Load Balancer**  
   - Configured HTTP frontend port 80 with an external IPv4 address.  
   - Backend points to `mig-flask-app` on port 8080, configured with health checks on `/health` endpoint.  
   - Utilizes `subnet-proxy-only` for proxy traffic.  

7. **Custom Domain Setup (Documented)**  
   - Created Cloud DNS public zone for the domain.  
   - Documented GoDaddy DNS delegation steps to Cloud DNS nameservers.  
   - Domain integration postponed due to cost.  

8. **Validation**  
   - Accessed load balancer external IP:  
     - `/` returns: `Hello from Flask on GCP (private subnet)!`  
     - `/health` returns HTTP 200 with `ok` body  

---

## Full Startup Script

#!/bin/bash
set -e

# Update & install prerequisites
apt-get update -y
apt-get install -y python3-pip python3-venv

# Create an app user if not present
id -u appuser &>/dev/null || useradd -m -s /bin/bash appuser

# Switch to appuser and set up a virtual environment
sudo -u appuser bash <<'EOF'
cd ~
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install flask gunicorn

# Create Flask app
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

# Create systemd service using venv's Gunicorn
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

# Enable and start service
systemctl daemon-reload
systemctl enable --now flask.service

Create systemd service for Gunicorn to serve Flask app
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

Enable and start the Flask systemd service
systemctl daemon-reload
systemctl enable --now flask.service


---

## Validation

- **Access Flask app frontend:**  
  Visit `http://<Load_Balancer_IP>/` in browser to see welcome message:  
  `"Hello from Flask on GCP (private subnet)!"`

- **Health check endpoint:**  
  Curl or visit `http://<Load_Balancer_IP>/health` returns:  
ok


---

## Why This Project?

- Demonstrates core Cloud and DevOps skills including secure VPC networking, private instance deployment, managed instance groups, and load balancing.
- Full automation of app deployment with a startup script eliminates manual server setup.
- Documents practical knowledge valuable for aspiring Cloud and DevOps engineers.
- Foundation for CI/CD, domain integration, and monitoring enhancements.

---

