# FP-TKA-Kelompok-2
## Langkah Implementasi
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
