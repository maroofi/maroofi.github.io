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



