GCP Private Flask App with Regional Load Balancer
Overview
This project demonstrates an end-to-end deployment of a secure, private Python Flask application on Google Cloud Platform (GCP), exposed via a Regional External HTTP(S) Load Balancer. The infrastructure includes a custom VPC network with multiple subnets, Cloud NAT for internet egress of private instances, firewall rules for security, a managed instance group running the Flask app via a startup script, and a planned custom domain setup with GoDaddy and Google Cloud DNS.

This project was implemented manually via the GCP web console for clarity and reproducibility, inspired by Abhishek's guide but fully designed and executed independently.

Architecture Diagram
![Architecture diagram for GCP Private Flask App with Regional Load Balancer](https://user-gen-media-assets.s3.amazonaws.com/gemini_images/eb8f2a04-17cb-4ae7-a7da- Key Technologies

Google Cloud Platform: VPC, Compute Engine, Managed Instance Groups (MIGs), Cloud NAT, Load Balancing (Regional External HTTP(S))

Python Flask: Lightweight web framework for the app

Gunicorn: WSGI HTTP server to serve Flask app

Startup Script: Automates installation and app launch on VMs

GoDaddy + Cloud DNS: For custom domain management (setup documented, pending activation)

Steps Implemented
1. Create Custom VPC and Subnets
Created a VPC named prod-vpc.

Defined three subnets within the same region (asia-south1):

subnet-public (10.10.10.0/24)

subnet-app (10.10.20.0/24) where app instances run

subnet-proxy-only (10.10.30.0/24) dedicated to load balancer proxy traffic

2. Set Up Cloud Router and Cloud NAT
Deployed Cloud Router (router-asia-south1) for dynamic routing.

Created Cloud NAT gateway (nat-asia-south1) attached to the router in the prod-vpc.

Allowed private VMs without public IPs to access the internet safely (for package installation and updates).

3. Firewall Rules Creation
Allowed health check traffic on TCP port 8080 from Google IP ranges (35.191.0.0/16, 130.211.0.0/22) to backend instances to enable load balancer health monitoring.

Allowed load balancer data-plane traffic on TCP 8080 exclusively from the subnet-proxy-only CIDR (10.10.30.0/24).

4. Instance Template and Startup Script
Created an instance template tmpl-flask-app with no external IP.

The startup script automates:

Installing Python3, pip, and virtual environment tools.

Creating a non-privileged user appuser.

Setting up a Python virtual environment with Flask and Gunicorn.

Creating the Flask application (app.py) with two routes: / and /health.

Configuring a systemd service flask.service to ensure the app runs on port 8080 reliably with Gunicorn.

5. Managed Instance Group (MIG)
Launched a managed instance group mig-flask-app in a single zone (asia-south1-a).

Autoscaling configured (min 2 instances, max 4) for load handling and resilience.

All instances run from the tmpl-flask-app instance template with private IPs only.

6. Regional External HTTP(S) Load Balancer
Configured Load Balancer with:

HTTP frontend on port 80 with a reserved external IPv4.

Backend pointing to the mig-flask-app instance group on port 8080.

Health checks targeting /health on backend instances.

Proxy-only subnet usage for the load balancer’s traffic.

Validated by accessing the LB external IP, confirming the Flask app responds properly on / and /health.

7. Domain Setup (Documented)
Created a Google Cloud DNS public zone for domain management.

Documented the process of delegating a GoDaddy domain to Cloud DNS with nameservers.

Did not enable live DNS due to cost concerns, but the entire flow is well recorded for future use.

8. Validation
Confirmed by browser and curl commands:

http://<Load_Balancer_IP>/ returns "Hello from Flask on GCP (private subnet)!"

http://<Load_Balancer_IP>/health returns HTTP 200 with body "ok".

Full Startup Script
#!/bin/bash
set -e

# Update system and install Python3 and venv
apt-get update -y
apt-get install -y python3-pip python3-venv

# Create 'appuser' user if not exists
id -u appuser &>/dev/null || useradd -m -s /bin/bash appuser

# Run as 'appuser': set up virtual env, install Flask and Gunicorn, and create Flask app
sudo -u appuser bash <<'EOF'
cd ~
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install flask gunicorn

# Write flask app code
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

# Create systemd service to run Gunicorn with 2 workers, bound on port 8080
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

# Enable and start the service
systemctl daemon-reload
systemctl enable --now flask.service

Why I Built This
To gain practical knowledge and confidence in configuring secure private networking and scalable app deployment on GCP.

Hands-on learning to illustrate key DevOps skills: infrastructure design, automation, load balancing, and domain integration.
