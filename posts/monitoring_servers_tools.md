---
layout: posts
---


## Tools and tips for monitoring servers

One of the useful approach to monitor several servers is to use **grafana** and **prometheus**. Here you can see the architecture of this approach:

```
+----------------+
| node_exporter 1|--+
+----------------+  |
+----------------+  |   scrapes    +--------------+   queries   +-------------+
| node_exporter 2|--+------------> |  Prometheus  | ----------> |  Grafana    |
+----------------+  |              | (stores time |             | (dashboards |
+----------------+  |              |  series data)|             | & alerts)   |
| node_exporter 3|--+              +--------------+             +-------------+
+----------------+
```

**node_exporters 1/2/3** are small agents which are installed directly on different machines. They generate stats about the servers. **prometheus** is the main tool which
collects the status from **node_exporters**, aggregate and store the stats and it also has its own query language to query the data afterwards. **Grafana** is used to create
dashboards and graphs from different data sources such as **prometheus**. What we see in grafana, we can also see it in Prometheus dashboard but it's ugly and text based. That's
why we install and use grafana. In this way, not only we can see things beautifully, but also we can connect different other data sources to grafana and have them all in on panel.

So the first step is to install node_exporter on different machines securely. 

### Installing node_exporter on Linux machine

Visiting node exporter [download page](https://prometheus.io/download/#node_exporter), we need to download the one that is suitable for our server, for example, if you use Linux machine,
_node_exporter-<VERSION>.linux-amd64.tar.gz_ is the right one. I have a test machine with Ubuntu installed and here is the steps:

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.12.1.linux-amd64.tar.gz
```

We make sure the hash is the exact match with the one on the dowload page:

```bash
sha256sum node_exporter-1.12.1.linux-amd64.tar.gz
# output b51d8a76aa2a9156a55d501aca6276fae09e262259a5e4e831d2c2222f084e63
# output is the exact match
```

extracting the tar file

```bash
tar -xvzf node_exporter-1.12.1.linux-amd64.tar.gz
cd node_exporter-1.12.1.linux-amd64
```
make sure the binary is working on your machine

```bash
./node_exporter --version
```
Now we will create a separate user without login for the node_exporter

```bash
sudo useradd --shell /usr/sbin/nologin pro_node_exporter
```
we create a `/etc/node_exporter` directory, move the binary to `/usr/local/bin` . From now on, we will work in `/etc/node_exporter` directory.
The binary and other related config files (we will create later), all belong to the new user we created.

```bash
# create the directory
sudo mkdir /etc/node_exporter

# move the binary
sudo mv node_exporter /usr/local/bin/node_exporter

# change the ownership of the binary
sudo chown pro_node_exporter:pro_node_exporter /usr/local/bin/node_exporter

```

We need to issue a certificate for connection (self-signed) and also create HTTP auth for our node exporter.
We will pass some config file to openssl to tell how to create a certificate. (Replace server IP address with <IP-ADDRESS>):
```bash
# content of the /etc/node_exporter/openssl.conf
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = node-exporter

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = <IP-ADDRESS>
```
Now we can issue a certificate by passing the config file:

```bash
sudo openssl req -x509 -nodes -days 825 -newkey rsa:2048   -keyout /etc/node_exporter/node_exporter.key   -out /etc/node_exporter/node_exporter.crt   -config /etc/node_exporter/openssl.conf

# now change the ownership of the key and crt file to the new user
sudo chown pro_node_exporter:pro_node_exporter /etc/node_exporter/node_exporter.key /etc/node_exporter/node_exporter.crt

# change the permission only to the owner
sudo chmod 600 /etc/node_exporter/node_exporter.key
sudo chmod 600 /etc/node_exporter/node_exporter.crt
```
We need to create a hash for HTTP auth for node_exporter (so that only those who have the password can see the metrics).

```bash
# install htpasswd if you don't have it: sudo apt install -y apache2-utils

# create the hash, it will ask for a password
htpasswd -nBC 10 "" | tr -d ':\n'
# I used this password: thisismypassword
# the result is something like: $2y$10$SIoEkqfqSvxK7jv3eD6FFOUgQOGIqhCMQCL7YQ//lDZ4CKaoDeU7a
```

Now create a config file `/etc/node_exporter/web-config.yml` and paste the followings:

```bash
tls_server_config:
  cert_file: /etc/node_exporter/node_exporter.crt
  key_file: /etc/node_exporter/node_exporter.key

basic_auth_users:
  prometheus: $2y$10$SIoEkqfqSvxK7jv3eD6FFOUgQOGIqhCMQCL7YQ//lDZ4CKaoDeU7a
```
Note that the username we chose is **prometheus** but you are free to choose whatever you want. Then, change the ownership of the `web-config.yml` file:

```bash
sudo chown pro_node_exporter:pro_node_exporter web-config.yml
sudo chmod 600 /etc/node_exporter/web-config.yml
```
Now it's time to run the `node_exporter` binary as a service in our Linux machine to make sure it's always working whenever we start the machine. We need to create a file
called `/etc/systemd/system/node_exporter.service` and add the following content:

```bash
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=pro_node_exporter
Group=pro_node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.config.file=/etc/node_exporter/web-config.yml

[Install]
WantedBy=multi-user.target
```
Let's reload the daemon, enable the service in every boot and check the status to make sure it's active.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl restart node_exporter
sudo service node_exporter status
```

If it's all good, we can see the service by using the following curl:

```bash
# -k is used because we have a self-signed certificate
curl -k -u prometheus:thisismypassword https://localhost:9100/metrics
# we must see a bunch of text on the screen
```
We should be able to also visit the page (in browser) by going to: `https://<YOUR-SERVER-PUBLIC-IP>:9100/metrics` (we will get certificate browser error at first).

**NOTE**: We can install the **node_exporter** on whatever number of machine we want, in the very same way I just explained. After each installation, we need to keep the 
1) public URL to visit the metric, 2) username and password we used for HTTP AUTH, 3) the generated hash for HTTP AUTH

Having all that installed, we can move on and install **Grafana** and **Prometheus** on one machine that is less busy and has enough disk or we can rent a VPS (small one) and install the 
tools there. Then, we will (securely) connect all the node_exporters to our dashboard.


### Installing Grafana and Prometheus (dockerized version)

It's easier to install **Grafana** and **Pormetheus** in docker. It's also possible to install it on the server directly but I don't see any advantage!

Things that we want to do are:
1. **Prometheus** panel won't be exposed on Internet. We want it only locally.
2. **Grafana** panel is exposed on Internet but we need a username and password for it.
3. **Grafana** panel must be HTTPS (secured) so we will issue a self-signed certificate for it.
4. Everything will work based on dockers but we need to make sure the data persist. So we'll store the data on the host.

We created 3 scripts as follows:
1. `generate-cert.sh`: will generate certificate for connecting to **Grafana** panel securely.
2. `prometheus.yml`: this is the configuration file we pass to **Prometheus**. We put all the information of the node_exporters here.
3. `docker-compose.yml`: this is our final docker file to run everything with docker compose.

first create a directory like `monitoring` for example so that we put everything in it.

Here is what we have in `generate-cert.sh`:
```bash
#!/bin/bash
# Generates a self-signed TLS certificate for Grafana.
# Replace CN / SAN with your actual server IP or hostname if you have one.

set -e

CERT_DIR="./certs"
DAYS_VALID=825   # ~2.25 years, common self-signed cert lifetime
CN="grafana.local"   # change to your domain or server IP

mkdir -p "$CERT_DIR"

openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout "$CERT_DIR/grafana.key" \
  -out "$CERT_DIR/grafana.crt" \
  -days $DAYS_VALID \
  -subj "/CN=$CN" \
  -addext "subjectAltName=DNS:$CN,IP:127.0.0.1"

# Grafana's container runs as UID 472 (non-root), not as whoever runs this
# script. 644 lets that user read the key when it's bind-mounted read-only.
# (Fine for a self-signed key used only by this local Grafana container;
# don't reuse this key/cert for anything more sensitive.)
chmod 644 "$CERT_DIR/grafana.key"
chmod 644 "$CERT_DIR/grafana.crt"

echo "Certificate generated in $CERT_DIR/"
echo "  - grafana.crt"
echo "  - grafana.key"
```

Here is the content of the `prometheus.yml` file:

```bash
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

# repeat this block for each server you installed node_exporter
# for example, if you have 10 servers, you need 10 blocks like this
  - job_name: 'srv-detection'         
    scheme: https
    tls_config:
      insecure_skip_verify: true
    basic_auth:
      username: prometheus      # here is your username for this node
      password: thisismypassword # here is the password you set on this node  
    static_configs:
      - targets: ['37.27.67.243:9100']   # <-- IP:PORT of the server
        labels:
          instance: 'srv-detection'          # <-- put a nice name here and we show this on grafana
  
````

And here is the `docker-compose.yml` file:
```bash
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    ports:
      # bind it to localhost to make not exposed on Internet
      # Access it via: ssh -L 9090:localhost:9090 user@IP-ADDR
      - "127.0.0.1:9090:9090"
    restart: unless-stopped
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./certs:/etc/grafana/certs:ro
    environment:
      # --- Auth ---
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin@123
      - GF_AUTH_ANONYMOUS_ENABLED=false
      - GF_AUTH_DISABLE_LOGIN_FORM=false
      # --- HTTPS ---
      - GF_SERVER_PROTOCOL=https
      - GF_SERVER_CERT_FILE=/etc/grafana/certs/grafana.crt
      - GF_SERVER_CERT_KEY=/etc/grafana/certs/grafana.key
      - GF_SERVER_DOMAIN=localhost
    ports:
      # This one IS meant to be reachable (dashboard), over HTTPS.
      - "3000:3000"
    restart: unless-stopped
    networks:
      - monitoring
    depends_on:
      - prometheus

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
```

The docker compose file will create a network bridge so that grafana and prometheus can connect to each other internally. 

Now that we have all 3 scripts, first run `generate-cert.sh` to generate the certificates. and then just simply run 
`docker compose up -d` to start the docker. Now we can visit `https://<IP-ADDRESS>:3000` to see the grafana panel.

**NOTES**: 
1. before running the docker, you need to change the value of `GF_SECURITY_ADMIN_PASSWORD` to something secure.
2. If you want to connect to prometheus dashboard (e.g., to see /targets), you can create a tunnel with ssh `ssh -L 9090:localhost:9090 user@<server-ip>` and then visit `http://localhost:9090` in you browser.
3. to set the data source in grafana, you must set it as `http://prometheus:9090` **not** `http://localhost:9090`. This is due to the fact that localhost inside the grafana docker means the grafana docker itself.
4. Make sure that you have docker in your user group: `sudo usermod -aG docker $USER` so that you **never** run docker with sudo.

The final step is to launch grafana panel (`https://<server-IP>:3000`) and start creating dashboards and panels to monitor servers.


--------------------------------------
