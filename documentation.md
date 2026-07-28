# DevOps Engineering Assignment: Real-Time WebSocket Chat Application

Deployment using Docker, Docker Compose, NGINX, AWS EC2 & GitHub Actions
Architecture Diagram
<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/6193c980-a91f-4967-8ecf-3f849ea7939f" />



                User Browser
                     │
                     │ HTTP / WebSocket
                     ▼
             AWS EC2 Public IP
                     │
                     ▼
         +----------------------+
         |   NGINX Container    |
         | Reverse Proxy        |
         | Serves Frontend      |
         +----------+-----------+
                    │
          WebSocket Proxy (/ws)
                    │
                    ▼
        +-----------------------+
        | FastAPI Backend        |
        | WebSocket Application  |
        +-----------------------+
Project Overview

The objective of this assignment was not to develop the backend application but to debug and fix an intentionally misconfigured deployment environment.

The repository already contained a working FastAPI WebSocket application, but the deployment configuration was broken. My responsibility was to identify the infrastructure issues, fix them, deploy the application on a cloud server, and automate the deployment using GitHub Actions.

The technologies used in this assignment include:

Docker
Docker Compose
NGINX Reverse Proxy
Docker Networking
FastAPI
AWS EC2
GitHub Actions (CI/CD)

# Step 1 – Cloning and Inspecting the Repository

I started by cloning the repository.

git clone https://github.com/yaswanthsai257/devops.git
cd devops

Before making any changes, I explored the project structure to understand how the application was organized.

The repository contained:

Dockerfile
docker-compose.yml
nginx.conf
FastAPI backend
Frontend HTML application

Instead of modifying the backend code, I focused only on the deployment configuration as required by the assignment.

# Step 2 – Running the Project

Following the assignment instructions, I attempted to build and run the application.

docker compose up -d --build

Initially, Docker failed while pulling the NGINX image due to a temporary Docker Hub authentication/network issue.

failed to resolve reference "docker.io/library/nginx:alpine"

failed to fetch oauth token

After verifying that Docker itself was working correctly by pulling images manually, I reran the build successfully and continued debugging the deployment.

# Step 3 – Understanding the Architecture

Before changing any configuration, I tried to understand how the application was expected to work.

The architecture is straightforward:

Browser
     │
     ▼
NGINX Reverse Proxy
     │
     ▼
FastAPI Backend
     │
     ▼
WebSocket Connection

Since the backend application itself was functioning correctly, the issues were likely in the infrastructure configuration.

The assignment specifically mentioned fixing only:

Dockerfile
docker-compose.yml
nginx.conf

Therefore, I focused only on these files.

# Step 4 – Fixing the Dockerfile

The first issue I noticed was in the Dockerfile.

Originally, Uvicorn was started using:

CMD ["uvicorn", "main:app", "--host", "127.0.0.1", "--port", "8000"]

Inside a Docker container, binding the application to 127.0.0.1 means that the server listens only inside that container.

Since NGINX runs in a different container, it would never be able to connect to the backend.

To allow communication over the Docker network, I changed it to:

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

Binding to 0.0.0.0 allows the application to accept connections from other containers on the Docker network.

# Step 5 – Fixing Docker Compose

Next, I inspected the Docker Compose configuration.

The backend service was configured correctly, and NGINX was exposing port 80.

However, I noticed that the frontend volume mapping had been commented out.

Originally:

# - ./frontend:/usr/share/nginx/html:ro

Because of this, the NGINX container had no frontend files available to serve.

I uncommented the volume mapping.

volumes:
  - ./frontend:/usr/share/nginx/html:ro
  - ./nginx.conf:/etc/nginx/nginx.conf:ro

I also verified that:

both containers were on the same Docker network
restart policies were enabled
backend was exposed internally
NGINX published port 80
Step 6 – Fixing NGINX

The final configuration issue was inside nginx.conf.

Originally the WebSocket proxy pointed to:

proxy_pass http://localhost:8000/ws;

Inside Docker, localhost refers to the NGINX container itself, not the backend container.

Since the backend runs in a different container, NGINX could never reach it.

I changed it to:

proxy_pass http://backend:8000/ws;

Docker Compose automatically creates internal DNS entries using service names, so backend correctly resolved to the FastAPI container.

Enabling WebSocket Upgrade

I also noticed that the WebSocket upgrade headers were commented out.

Originally:

# proxy_set_header Upgrade $http_upgrade;
# proxy_set_header Connection "upgrade";

Without these headers, HTTP requests cannot be upgraded to WebSocket connections.

I uncommented them.

proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";

This completed the reverse proxy configuration.

# Step 7 – Testing the Application Locally

After fixing all three configuration files, I rebuilt the project.

docker compose up -d --build

The containers started successfully.

docker ps

showed both:

chat-backend
chat-nginx

running correctly.

I also verified the Docker network.

docker network inspect devops_default

Everything was connected correctly.

Finally, I opened:

http://localhost

The application loaded successfully.

This confirmed that:

NGINX was serving the frontend
requests were reaching the backend
Docker networking was working correctly
WebSocket proxying was functioning

I then opened multiple browser tabs and exchanged messages between them.

Messages appeared instantly across all connected clients, confirming that the real-time WebSocket functionality was working correctly.

(Insert screenshot here.)

# Step 8 – Deploying to AWS EC2

Once the application worked locally, I deployed it to AWS EC2 using the Free Tier.

After launching the instance, I connected using SSH.

ssh -i key.pem ubuntu@<public-ip>

I then installed Docker.

sudo apt update

sudo apt install docker.io -y

sudo systemctl enable docker

sudo systemctl start docker

Docker Compose was already available on the instance.

To avoid using sudo for every Docker command, I added the Ubuntu user to the Docker group.

sudo usermod -aG docker ubuntu

After reconnecting to the server, Docker commands worked without sudo.

# Step 9 – Deploying the Project

I cloned my repository onto the EC2 instance.

git clone https://github.com/TriptiP-Code/spotsure_biz_assignment.git

cd spotsure_biz_assignment

Then started the application.

docker compose up -d --build

The containers started successfully.

Initially, the application was still inaccessible from outside the server.

The issue was not Docker but the AWS Security Group.

I added an inbound rule allowing HTTP traffic.

Type : HTTP

Port : 80

Source : 0.0.0.0/0

After updating the security group, the application became publicly accessible.

http://13.233.117.28
Step 10 – Automating Deployment using GitHub Actions

The final task was implementing continuous deployment.

I created a GitHub Actions workflow that automatically deploys the latest version of the application whenever code is pushed to the main branch.

The workflow performs the following steps:

Trigger on every push to main
Connect to the EC2 instance using SSH
Pull the latest code
Stop existing containers
Rebuild Docker images
Restart the application
Remove unused Docker images
Configuring Secure SSH Authentication

GitHub Actions runs on GitHub-hosted runners, not on my local machine.

Therefore, it needed a secure way to connect to my EC2 instance.

I generated a dedicated SSH key pair on the EC2 server.

ssh-keygen -t ed25519 -C "github-actions"

This created:

~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub

The private key (id_ed25519) was stored securely as a GitHub Repository Secret named:

EC2_SSH_KEY

The public key (id_ed25519.pub) was appended to:

~/.ssh/authorized_keys

This allowed the EC2 server to trust SSH connections initiated by GitHub Actions.

I also configured the following GitHub Secrets:

EC2_HOST
EC2_USER
EC2_SSH_KEY
Initial CI/CD Issue

The first workflow execution failed with:

ssh: unable to authenticate

The reason was that although I had generated the SSH key pair, the public key had not yet been added to the server's authorized_keys file.

After running:

cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

the EC2 server trusted GitHub Actions.

The next workflow completed successfully.

Every push to the repository now automatically deploys the latest version of the application without any manual intervention.

Outcome

By the end of the assignment, I had successfully:

Fixed the Docker configuration
Corrected Docker Compose setup
Configured Docker networking
Configured NGINX as a reverse proxy
Enabled WebSocket proxying
Verified multi-user real-time chat functionality
Deployed the application on AWS EC2
Configured GitHub Actions for automated deployment
Implemented a production-style CI/CD pipeline

This assignment provided hands-on experience with debugging containerized applications, configuring reverse proxies, managing Docker networking, deploying applications to the cloud, and automating deployments using GitHub Actions. It closely resembled the responsibilities involved in real-world DevOps workflows.
