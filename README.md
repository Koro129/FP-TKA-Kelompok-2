# Final Project TKA - Kelompok 2 #

## Anggota : ##
- Fathika Afrine A. (5027211016)
- Andreas Timotius P. S. (5027211019)
- Fransiskus Benyamin S. (5027211021)
- Muh. Ilham Yumna (5027211024)
- Arkan Hendri (5027211026)

# Laporan Final
## Daftar Isi
- [Laporan Final](#laporan-final)
  - [Daftar Isi](#daftar-isi)
  - [Introduction](#introduction)
  - [Rancangan Arsitektur](#rancangan-arsitektur)
  - [Tabel Spesifikasi](#tabel-spesifikasi)
  - [Langkah Implementasi](#langkah-implementasi)
    - [Load Balancing](#load-balancing)
    - [Instalasi app.py](#instalasi-apppy)
  - [Hasil Pengujian Endpoint](#hasil-pengujian-endpoint)
    - [Get /orders](#get-orders)
    - [GET /orders/<order_id>](#GET-ordersorder_id)
    - [POST /orders](#POST-orders)
    - [PUT /orders/<order_id>](#PUT-ordersorder_id)
    - [DELETE /orders/<order_id>](#DELETE-ordersorder_id)

## Rancangan Arsitektur
![Final Project Cloud-Page-2 drawio](https://github.com/Koro129/FP-TKA-Kelompok-2/assets/88912492/c96461ad-c7d5-41f2-9d4b-1da67882c210)

## Tabel Spesifikasi
| No | Nama | Spesifikasi | Fungsi | Harga/bulan |
|----|-------|--------------------------------|-------------------|-----|
| 1 | Aura | Regular 1vCPU, 2gb Memory | Load Balancer | 12$ |
| 2 | Dynasty |	Premium AMD 1vCPU, 2gb Memory | App Worker 1 | 14$ |
| 3 | Trove | Premium AMD 1vCPU, 2gb Memory | App Worker 2 | 14$ |
| 4 | Legion | Premium AMD 1vCPU, 2gb Memory | App Worker 3 | 14$ |
| 5 | Himmel | Regular 1vCPU, 2gb Memory | Database Server | 12$ |
|||| Total | 66$ |

## Langkah Implementasi
### Setup database
Pertama, kita perlu membuat bash file berisikan script berikut pada server database untuk menginstall mongodb

```bash
#!/bin/bash

# Install required packages
sudo apt-get install -y gnupg curl

# Download and save MongoDB GPG key
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor

# Add MongoDB repository to sources list
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Update package list
sudo apt-get update

# Install MongoDB
sudo apt-get install -y mongodb-org

# Start MongoDB service
sudo systemctl start mongod

# Enable MongoDB service to start on boot
sudo systemctl enable mongod

# Display MongoDB service status
sudo systemctl status mongod
```

buat scriptnya executable:

```bash
chmod +x install_mongodb.sh
```

lalu jalankan scriptnya:

```bash
./install_mongodb.sh
```

lalu kita membuat `update_nftables.sh` untuk memperbolehkan akses ssh dari port 22, serta membuka akses untuk worker yang akan mengakses database

```bash
#!/bin/bash

# Define the path to the nftables configuration file
NFTABLES_CONF="/etc/nftables.conf"

# Backup the current nftables configuration file
cp "$NFTABLES_CONF" "$NFTABLES_CONF.bak"

# Create a new nftables configuration
cat <<EOF > "$NFTABLES_CONF"
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
        chain input {
                type filter hook input priority 0; policy drop;

                # Allow loopback traffic
                iif "lo" accept

                # Allow incoming traffic on MongoDB port from specific IP addresses
                tcp dport 27017 ip saddr { 178.128.22.10, 165.232.171.179, 68.183.237.141 } accept
                tcp dport 22 accept
        }
        chain forward {
                type filter hook forward priority 0;
        }
        chain output {
                type filter hook output priority 0;
        }
}
EOF

echo "Updated $NFTABLES_CONF with new rules."

# Reload nftables to apply the changes
sudo nft -f "$NFTABLES_CONF"

echo "nftables reloaded successfully."
```

buat jadi executable:

```bash
chmod +x update_nftables.sh
```

lalu jalankan scriptnya:

```bash
sudo ./update_nftables.sh
```

lalu kita akan membuat `update_mongod_conf.sh` untuk membuka akses db dan memasang otorisasi seperti dibawah ini:

```bash
#!/bin/bash

# Define the path to the mongod configuration file
MONGOD_CONF="/etc/mongod.conf"

# Backup the current mongod configuration file
cp "$MONGOD_CONF" "$MONGOD_CONF.bak"

# Create a new mongod configuration
cat <<EOF > "$MONGOD_CONF"
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /var/lib/mongodb
#  engine:
#  wiredTiger:

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0

# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

#security:
security:
  authorization: enabled

#operationProfiling:

#replication:

#sharding:

## Enterprise-Only Options:

#auditLog:
EOF

echo "Updated $MONGOD_CONF with new configuration."

# Restart the mongod service to apply the changes
sudo systemctl restart mongod

echo "mongod service restarted successfully."
```

buat scriptnya menjadi executable:

```bash
chmod +x update_mongod_conf.sh
```

lalu jalankan scriptnya:

```bash
sudo ./update_mongod_conf.sh
```

lalu kita buat juga `create_mongo_user.sh` untuk membuat akun dimana worker dapat memodifikasi database:

```bash
#!/bin/bash

# Connect to MongoDB using mongosh
mongosh <<EOF
use admin
db.createUser({
  user: "worker",
  pwd: "Kelompok2TKA",
  roles: [
    { role: "readWrite", db: "orders_db" },
  ]
})
EOF
```

buat scriptnya menjadi executable:

```bash
chmod +x create_mongo_user.sh
```

lalu jalankan scriptnya:

```bash
./create_mongo_user.sh
```

### Load Balancing
jalankan script berikut pada load balancer
```
apt-get update
apt-get install nginx -y

echo '
upstream appfp  {
	least_conn;
	server 178.128.22.10:5000; #Worker1
	server 165.232.171.179:5000; #Worker2
	server 68.183.237.141:5000; #Worker3
}

server {
 	listen 80;
 	server_name _;

 	location / {
 		proxy_pass http://appfp;
 	}
}' > /etc/nginx/sites-available/appfp

rm -rf /etc/nginx/sites-enabled/default

ln -s /etc/nginx/sites-available/appfp /etc/nginx/sites-enabled

service nginx restart
```
### Instalasi app.py
Jalankan script berikut untuk menyiapkan app.py pada tiap worker
```
if [ -e app.py ]; then
    rm app.py
    echo "File app.py yang sudah ada telah dihapus."
fi

# Install Python3
apt-get update
apt-get install python3 -y

# Install pip for Python3
apt-get install python3-pip -y

# Install pymongo and flask using pip
pip3 install pymongo flask
pip3 install Flask-PyMongo

# Download app.py from the given GitHub repository
wget https://raw.githubusercontent.com/fuaddary/fp-tka/main/app.py

# konfigurasi MongoDB URI ke IP DB
sed -i "s|app.config\['MONGO_URI'\] = 'mongodb://localhost:27017/orders_db'|app.config['MONGO_URI'] = 'mongodb://worker:Kelompok2TKA@167.71.217.115:27017/orders_db?authSource=admin'|" app.py

# Konfigurasi agar berjalan pada host 0.0.0.0 dan port 5000
sed -i 's/app.run(debug=True)/app.run(host="0.0.0.0", port=5000)/' app.py
```  
dan script berikut untuk menjalankan aplikasi di background
```
# Mendapatkan ID proses yang berjalan pada port 5000
PID=$(lsof -t -i :5000)

# Memeriksa apakah proses dengan ID tersebut sedang berjalan
if [ -n "$PID" ]; then
    # Jika berjalan, membunuh proses tersebut
    echo "Membunuh proses dengan ID: $PID"
    kill -9 $PID
fi

# Menjalankan ulang app.py di background
nohup python3 app.py > app.log 2>&1 &
echo "Aplikasi dijalankan ulang."
```
## Hasil Pengujian Endpoint
### Get /orders
![image](https://github.com/Koro129/FP-TKA-Kelompok-2/assets/113784446/1f04dc46-0047-474b-b6a3-a4af0a765855)
### GET /orders/<order_id>
![image](https://github.com/Koro129/FP-TKA-Kelompok-2/assets/113784446/0a64ca28-f5eb-4358-99b6-d2cf6bd2c006)
### POST /orders
![image](https://github.com/Koro129/FP-TKA-Kelompok-2/assets/113784446/d33f444e-1a72-4b7c-9a76-56c0925dd141)
### PUT /orders/<order_id>
![image](https://github.com/Koro129/FP-TKA-Kelompok-2/assets/113784446/1896ae22-fd4c-4844-87a6-28d317309b85)
### DELETE /orders/<order_id>
![image](https://github.com/Koro129/FP-TKA-Kelompok-2/assets/113784446/847616ce-96e9-4259-a700-586378b55020)



