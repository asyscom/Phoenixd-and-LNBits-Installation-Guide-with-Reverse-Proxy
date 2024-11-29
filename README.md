---

# Phoenixd and LNBits Installation Guide with Reverse Proxy

## Introduction
This guide explains how to set up **Phoenixd** (version `v0.3.2`) and **LNBits** (version `v0.4.2`), including integration with PostgreSQL and a reverse proxy using **Nginx**. Phoenixd simplifies Lightning Network node management, and LNBits provides a powerful web interface for managing your Lightning funds.

---

## Prerequisites
Ensure you have the following installed on your system:
- **Ubuntu 20.04/22.04**
- **Root or sudo privileges**

---

## 1. Install Necessary Tools
```bash
sudo apt-get update
sudo apt-get install -y unzip wget software-properties-common
```

---

## 2. Install Phoenixd
Download the latest Phoenixd release:
```bash
wget https://github.com/ACINQ/phoenixd/releases/download/v0.3.2/phoenix-0.3.2-linux-x64.zip
unzip phoenix-0.3.2-linux-x64.zip && rm -r phoenix-0.3.2-linux-x64.zip
cd phoenix-0.3.2-linux-x64
```

Run Phoenixd:
```bash
./phoenixd --agree-to-terms-of-service
```

You should see an output like this:
```
2024-07-27 10:26:12 datadir: /home/users/.phoenix
2024-07-27 10:26:12 chain: Mainnet
2024-07-27 10:26:12 listening on http://127.0.0.1:9740
```

### Verify Phoenixd
In another terminal, navigate to the Phoenixd directory:
```bash
./phoenix-cli getinfo
```

---

## 3. Set Up Phoenixd as a Systemd Service
Create a systemd service file for Phoenixd:
```bash
sudo nano /etc/systemd/system/phoenixd.service
```

Add the following:
```ini
[Unit]
Description=Phoenixd
After=network.target

[Service]
ExecStart=/home/<your_user>/phoenix-0.3.2-linux-x64/phoenixd --agree-to-terms-of-service
Restart=always
User=<your_user>

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl enable phoenixd.service
sudo systemctl start phoenixd.service
sudo systemctl status phoenixd.service
```

---

## 4. Install LNBits (v0.4.2)

### 4.1 Install Python and Poetry
Add the Python PPA and install Python:
```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt install python3.9 python3.9-distutils
```

Install Poetry:
```bash
curl -sSL https://install.python-poetry.org | python3 -
```

Set up Poetry's path:
```bash
export PATH="/home/<your_user>/.local/bin:$PATH"
```

To make this change permanent:
```bash
echo 'export PATH="/home/<your_user>/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 4.2 Clone and Install LNBits
Clone the LNBits repository:
```bash
git clone https://github.com/lnbits/lnbits.git
cd lnbits
git checkout v0.4.2
```

Install dependencies:
```bash
poetry env use python3.9
poetry install --only main
mkdir data
```

---

## 5. Configure PostgreSQL for LNBits
Install PostgreSQL:
```bash
sudo apt-get install -y postgresql
sudo -i -u postgres
```

Set up the database:
```sql
ALTER USER postgres PASSWORD 'StrongPassword!';
createdb lnbits;
exit
```

---

## 6. Configure LNBits
Create a `.env` file in the LNBits directory:
```bash
nano .env
```

Add the following:
```env
LNBITS_ADMIN_UI=true
HOST=yourdomain.com
FORWARDED_ALLOW_IPS="*"
PORT=5000
LNBITS_DATABASE_URL=postgres://postgres:StrongPassword!@localhost:5432/lnbits
PHOENIXD_API_ENDPOINT=http://127.0.0.1:9740/
PHOENIXD_API_PASSWORD=<your_phoenixd_password>
```

Start LNBits:
```bash
poetry run lnbits --port 5000 --host 0.0.0.0
```

---

## 7. Run LNBits as a Systemd Service
Create a systemd service file for LNBits:
```bash
sudo nano /etc/systemd/system/lnbits.service
```

Add the following:
```ini
[Unit]
Description=LNbits

[Service]
WorkingDirectory=/home/<your_user>/lnbits
ExecStart=/home/<your_user>/.local/bin/poetry run lnbits --port 5000 --host 0.0.0.0
User=<your_user>
Restart=always
TimeoutSec=120
RestartSec=30
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl enable lnbits.service
sudo systemctl start lnbits.service
sudo systemctl status lnbits.service
```

---

## 8. Set Up a Reverse Proxy with Nginx
### 8.1 Install Nginx and Certbot
```bash
sudo apt-get install -y nginx certbot python3-certbot-nginx
```

### 8.2 Configure Nginx
Create an Nginx configuration file:
```bash
sudo nano /etc/nginx/sites-available/lnbits
```

Add the following:
```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
    }
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the configuration:
```bash
sudo ln -s /etc/nginx/sites-available/lnbits /etc/nginx/sites-enabled/
sudo certbot --nginx -d yourdomain.com
sudo systemctl reload nginx
```

---

## 9. Test Your Setup
Access LNBits at `https://yourdomain.com`, create a SuperUser account, and start using your Phoenixd-connected LNBits instance!

🎉 Congratulations! Your setup is now complete.
