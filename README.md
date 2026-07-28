# DevOps Engineering Assignment: Real-Time Chat App  SUBMISSION 

### Q1

When you run

```
node server.js
```

who actually starts the program?
windows run the program 

---

### Q2

Is VS Code running your backend?

Or is it just helping you?

Explain.
no vs code only sends the command run node.js , and then windows run the node.exe file
so vs code is only a platform to write the command 

---

### Q3

What is a process?

Give **3 examples** besides Node.
Process are the task runninng on the OS , it is scheduled by cpu
other examples are chrome , vs code , asus application

---

### Q4

Why does

> "It works on my machine"

happen?
because may be the person who created the application is running it in a machine having compatible node version , dependncies or env which is suitable to run the application but may be another person has a diff env i.e another node version , another os then application will give error 

---

### Q5

If Windows manages CPU, RAM and processes,

why do we even need Docker?
may be docker will create an image which can run on any machine having docker , when they do docker run a container will get created using the image

(Don't worry if you can't answer this completely yet. Just think about it.)

===============================================================

### Q1

What is virtualization?
in one big server or system , it can run multiple machines (vm) i.e in one server , there can be windows , linux , centos . each os has their own machine , their own ram , kernal and all machines are independent of each other and in those machine we can run node appplication. This genrally help us to use the resource efficently , i.e no need of having 3 servers for 3 diff team , get one big server and user hypervisor and vitualise it

---

### Q2

What is a Hypervisor?
hypervisor is installed in a machine and makes  vm ,on top of server 

---

### Q3

Why does every Virtual Machine need its own Operating System?
because they r seperate machine , although on top of a server but acting as an indepenednt machine , so whatever os is best for our application we can install it

---

### Q4

If we create 100 Virtual Machines,

what gets duplicated 100 times?
i dont know

---

### Q5 ⭐ (Most Important)

Why did engineers start looking for an alternative to Virtual Machines?
because of space issue , in virtualisation the ram , cpu are equally distributed on the system , so even if the application need more space , we cannnot give it

---

============================================
PROJECT

I cloned the project
 git clone https://github.com/yaswanthsai257/devops.git

the inspected the files 

and then did
docker compose up -d --build

got below error

devops on  main on ☁️  (us-east-1) 
❯ docker compose up -d --build
time="2026-07-28T15:46:34+05:30" level=warning msg="C:\\Users\\tript\\Desktop\\PROJECTS\\Spotsure_biz\\devops\\docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"
[+] Running 0/1
[+] Running 0/1g                                                                                                                   
[+] Running 1/1g                                                                                                                   
 ✘ nginx Error failed to resolve reference "docker.io/library/nginx:alpine": failed to authorize: failed to f...             26.7s 
Error response from daemon: failed to resolve reference "docker.io/library/nginx:alpine": failed to authorize: failed to fetch oauth token: Post "https://auth.docker.io/token": EOF

Now I have 1st changes docker file 
from 
CMD ["uvicorn", "main:app", "--host", "127.0.0.1", "--port", "8000"]

to
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

Then changed the docker compose file
uncommented - ./frontend:/usr/share/nginx/html:ro

ubuntu@ip-172-31-7-216:~/spotsure_biz_assignment$


![[Pasted image 20260728191012.png]]




=================================

DevOps Engineering Assignment Deployment of Real-Time WebSocket Application (Docker + Nginx + CI/CD)


First I started by cloning the repository and inspected the project structure.

git clone https://github.com/yaswanthsai257/devops.git  
cd devops

then found that there are many files like Dockerfile , docker-compose.yml , nginx.conf , FastAPI backend , Frontend HTML page

I had already docker desktop in the my laptop so as instructed  I ran:

docker compose up -d --build

docker was unable to pull the required images because of a temporary Docker Hub authentication/network issue. 

devops on  main on ☁️  (us-east-1) 
❯ docker compose up -d --build
time="2026-07-28T15:46:34+05:30" level=warning msg="C:\\Users\\tript\\Desktop\\PROJECTS\\Spotsure_biz\\devops\\docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"
[+] Running 0/1
[+] Running 0/1g                                                                                                                   
[+] Running 1/1g                                                                                                                   
 ✘ nginx Error failed to resolve reference "docker.io/library/nginx:alpine": failed to authorize: failed to f...             26.7s 
Error response from daemon: failed to resolve reference "docker.io/library/nginx:alpine": failed to authorize: failed to fetch oauth token: Post "https://auth.docker.io/token": EOF

Now I have 1st changes docker file 
from 
CMD ["uvicorn", "main:app", "--host", "127.0.0.1", "--port", "8000"]

to
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]


Now before changing anything , I went through the file structure and tried to understand the application 

Using my brain and experience I found that 

Browser ---> NGINX (r proxy) ---> Backend --->Websocket

As mentioned in assignment I did not tried to touch BE code , but focused on infra 
There were 3 config file docker , docker compose , nginx file

Started with Docker file

the application was starting FastAPI using  uvicorn main:app --host 127.0.0.1 --port 8000 inside a Docker container, binding the application to **127.0.0.1** means that the server is only accessible from inside that container

since NGINX is running in a separate container, it would never be able to connect to the backend.

to make the backend accessible to other containers on the Docker network, I changed it to:
CMD ["uvicorn","main:app","--host","0.0.0.0","--port","8000"]

this allowed the backend service to listen on all network interfaces inside the container.


Now comes docker compose , the backend container was being built correctly, and the NGINX container was exposing port 80.
however, I noticed that the frontend volume mapping was commented out.

originally

 # - ./frontend:/usr/share/nginx/html:ro
this means that NGINX had no frontend file to serve 

I uncommented the volume mapping 

volumes:
  - ./frontend:/usr/share/nginx/html:ro
  - ./nginx.conf:/etc/nginx/nginx.conf:ro

now the frontend files would be available inside the Nginix container

I also verified that , both services were on the same docker network , restart policy was enabled ,backend was exposed internally and  NGINX published port 80


After docker things , now comes NGINX 
I observed web socket proxy was configured as 
proxy_pass http://localhost:8000/ws;

inside Docker, localhost refers to the Nginix container itself , not the backend container
since the backend runs in another container, Nginix could not reach it.
I changed it to 
proxy_pass http://backend:8000/ws;

docker compose automatically creates DNS entries using the service names, so "backend" correctly resolved to the FastAPI container.

then i found that the required WebSocket upgrade headers were commented out.

Originally :

##proxy_set_header Upgrade $http_upgrade;
##proxy_set_header Connection "upgrade";

Without these headers, WebSocket connections cannot be upgraded from HTTP I uncommented them.

The files were correctly mounted 
root /usr/share/nginx/html;

Now its time to test 
So ran docket build

docker compose up -d --build

❯ docker ps
CONTAINER ID   IMAGE            COMMAND                  CREATED       STATUS       PORTS                                 NAMES
fb75a6b84154   nginx:alpine     "/docker-entrypoint.…"   4 hours ago   Up 4 hours   0.0.0.0:80->80/tcp, [::]:80->80/tcp   chat-nginx
968f94285809   devops-backend   "uvicorn main:app --…"   4 hours ago   Up 4 hours   8000/tcp                              chat-backend


After checking the networking 
docker network inspect devops_default
everything was correct 
than simply ran 
http://localhost

hurrayy its working now , I can see the 

![[Pasted image 20260728205619.png]]


This confirmed that Nginix successfully proxied the request ,  WebSocket upgrade headers were working ,  FastAPI accepted the connection

I then opened multiple browser tabs and exchanged messages between them.

Messages were instantly broadcast to all connected clients, confirming that the real-time chat functionality was working correctly.

Below is the Example
![[Pasted image 20260728205930.png]]





Now comes EC2 , Used free tier 

Launched ec2  and connected using ssh

ssh -i key.pen ubuntu@public-ip

then in the instance downloaded docker

sudo apt install docker.io -y

sudo systemctl enable docker
sudo systemctl start docker

installed docker compose 
(here got many error fixed this )

then did
sudo usermod -aG docker ubuntu

so that i dont have to use sudo all time

then did git clone https://github.com/TriptiP-Code/spotsure_biz_assignment.git

got my files in the instance 

docker compose up -d --build 

and now when I did 

![[Pasted image 20260728210720.png]]

Although had some networking issue , updated the security group  added an inbound rule ,Type: HTTP ,Port: 80 , Source: 0.0.0.0/0
then only http://13.200.243.83/ worked


Now comes the CI/CD github action always my fav 

Created a workflow

1st to make ensure the runner connectiion
when GitHub Actions runs,  it executes on a GitHub-hosted runner not on my laptop.
when the workflow reaches : 

name: Deploy to EC2
uses: appleboy/ssh-action@v1.2.0

GitHub needs a secure way to log in to your EC2 instance ,since there is no person to type a password, we use SSH key authentication.
1st  Generate a new SSH key pair on EC2

On the EC2 server, we ran:

ssh-keygen -t ed25519 -C "github-actions"

This generated two files inside:

~/.ssh/

they were:

id_ed25519
id_ed25519.pub

1. Private Key
~/.ssh/id_ed25519

Secret , to never shared publicly ,used by GitHub Actions to prove its identity

I copied the entire contents of this file into the GitHub secret :

EC2_SSH_KEY
2. Public Key
~/.ssh/id_ed25519.pub

 Then Add the public key to authorized_keys

initially, EC2 server only trusted  AWS key pair (the .pem key you use from my laptop).

So I ran:

cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

Now the file looked like :
authorized_keys
AWS public key
GitHub Actions public key

and this means the EC2 server now trusts both keys.

Then storing the private key in GitHub Secrets

in GitHub, there is secrets and variable option they save the secrets

created a secret :

EC2_SSH_KEY and pasted the contents of :

~/.ssh/id_ed25519 (the private key).

I also added secrets like :

EC2_HOST
EC2_USER
EC2_SSH_KEY

GitHub Actions uses the key when code is pushed i.e

git push origin main

GitHub Actions started and it executed :

uses: appleboy/ssh-action@v1.2.0

This action: 
Reads EC2_HOST
Reads EC2_USER
Reads EC2_SSH_KEY
Creates an SSH connection to EC2
Executes the deployment commands

Now first time

The workflow failed with:

ssh: unable to authenticate

because although I had generated the key pair, I had not yet added the public key to authorized_keys.

The EC2 server didn't recognize GitHub's key.

The fix I ran :

cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

After that : Github private key matched with auth key


And then the Pipleine ran , You can see the github actions




