# AWS EC2 Assessment 1: SRE Drill for EC2 Performance Degradation  

The application exposes one endpoint:

```text
GET /health
```

When EC2 system health is degrade, the endpoint returns HTTP `503`:

```json
{"status":"unhealthy"}
```

## AWS specifications

| Resource         | Specification           |
| ---------------- | ----------------------- |
| EC2 instance     | `t2.micro`              |
| Operating system | Ubuntu Server 22.04 LTS |
| Root EBS volume  | 10 GiB                  |
| EBS volume type  | `gp3`                   |
| Public IPv4      | Enabled                 |
| Uvicorn workers  | 1                       |

### Security Group

| Port   | Source                                       | Purpose    |
| ------ | -------------------------------------------- | ---------- |
| TCP 22 | Student public IP                            | SSH        |
| TCP 80 | Student public IP or required client network | Nginx HTTP |

Do not expose port `3000` publicly. Uvicorn should listen only on `127.0.0.1`, with Nginx configured as a reverse proxy.

## Deployment

Connect to EC2:

```bash
chmod 400 <custom-private-key>.pem
ssh-add <custom-private-key>.pem
ssh ubuntu@EC2_PUBLIC_IP
```

Install the required packages:

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git nginx
```

Clone the repository:

```bash
git clone REPOSITORY_URL
cd REPOSITORY_DIR
```

Create the log directory:

```bash
sudo mkdir -p /var/log/storage-breaker
sudo chown -R ubuntu:ubuntu /var/log/storage-breaker
```

Install the Python dependencies:

```bash
cd /opt/storage-breaker

python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install -r requirements.txt
```

Start the application:

```bash
cd /opt/storage-breaker

.venv/bin/uvicorn app:app \
  --host 127.0.0.1 \
  --port 3000 \
  --workers 1 \
  --no-access-log
```
Another option to run app
```bash
cd /opt/storage-breaker/ape-aws-ec2-assessment-1
/opt/storage-breaker/.venv/bin/uvicorn app:app \
  --host 127.0.0.1 \
  --port 3000 \
  --workers 1 \
  --no-access-log
```
 or 
 ```bash
  ../.venv/bin/uvicorn app:app --host 127.0.0.1 --port 3000 --workers 1 --no-access-log
  ```
Configure Nginx as a reverse proxy from port `80` to:

```bash
http://127.0.0.1:3000
```

Test the application locally:

```bash
curl -i http://127.0.0.1:3000/health
```

Test through Nginx:

```bash
curl -i http://EC2_PUBLIC_IP/health
```

Do not use more than one Uvicorn worker. Each worker starts an additional log-generation thread and increases the disk-consumption rate.

List of the storage issues when configure the reversed proxy server :

```bash
ubuntu@ip-172-31-20-0:/opt/storage-breaker/ape-aws-ec2-assessment-1$ sudo apt-get install nginx -y
Reading package lists... Error!
E: Write error - write (28: No space left on device)
E: IO Error saving source cache
E: The package lists or status file could not be parsed or opened.
```
Step 1-Find the process 
```bash
ps aux | grep app.py
```
Step 2 - Kill it 
```bash
sudo kill -9 <PID>
```
Or kill it in one command:
```bash
sudo pkill -f app.py
```
Step 3 — Verify it's stopped
```bash
ps aux | grep app.py
```
Step 4 — Truncate the log
```bash
sudo truncate -s 0 /var/log/storage-breaker/application.log
```

Step 5 — check free space
```bash
df -h /dev/root
```
Install Ngnix
```bash
Sudo apt-get install nginx -y
```
 Create nginx reverse proxy config
```bash
buntu@ip-172-31-20-0:/opt/storage-breaker/ape-aws-ec2-assessment-1$ sudo tee /etc/nginx/sites-available/storage-breaker > /dev/null << 'EOF'
server {
    listen 80;
    server_name 54.91.181.1;

    location /health {
        proxy_pass http://127.0.0.1:3000/health;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF
```
Enable and restart:
```bash
sudo ln -s /etc/nginx/sites-available/storage-breaker /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```
Result 
```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Test with public :
```bash
curl http://54.91.181.1/health  #public_IP 
```
Result:
```bash
{"status":"healthy"}ubuntu@ip-172-31-20-0:/opt/storage-breaker/ape-aws-ec2-assessment-1$ 
```
