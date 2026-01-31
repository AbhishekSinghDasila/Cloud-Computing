# 🌐 Distributed Cloud Calculator (NGINX + Flask on KVM VMs)

This project demonstrates a **distributed cloud application** using two Virtual Machines:

- **VM-WEB** → Frontend using NGINX
- **VM-REST** → Backend REST Calculator using Flask

The frontend sends requests to the backend over the network, simulating **microservice architecture** in a cloud environment.

---

## 🏗 Architecture Overview
```
    Host Browser
          |
          v
   +---------------+
   |   VM-WEB      |
   |   NGINX       |
   |   Frontend    |
   +---------------+
          |
       HTTP API
          |
          v
   +---------------+
   |   VM-REST     |
   | Flask REST API|
   | Calculator    |
   +---------------+
```

Flow:

Browser → NGINX (vm-web) → Flask API (vm-rest) → Result → Browser

---

## ⚙️ Environment Used

- Ubuntu 20.04 VMs
- KVM / QEMU virtualization
- NGINX web server
- Python Flask REST API
- Flask-CORS
- uWSGI
- Virtual environment (venv)

---

## 🖥 VM Setup

Two VMs required:

| VM | Role | Example IP |
|----|------|-----------|
| vm-web | Frontend | 192.168.122.134 |
| vm-rest | Backend | 192.168.122.90 |

---

## 🔹 Step 1 – Identity Setup

## WM-REST
```bash
sudo hostnamectl set-hostname vm-rest
exec bash
```

### Install dependencies 
```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv build-essential -y
```

### creating Virtual Environment
```bash
mkdir -p ~/lab1
cd ~/lab1
python3 -m venv venv
source venv/bin/activate
pip install flask flask-cors uwsgi
```

### Creating Flask Calculator Api
```
nano ~/lab1/app.py

from flask import Flask, request
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

@app.route('/calculate')
def calculate():
    a = float(request.args.get('a', 0))
    b = float(request.args.get('b', 0))
    op = request.args.get('op', 'add')

    if op == 'add':
        return str(a + b)
    elif op == 'sub':
        return str(a - b)
    elif op == 'mul':
        return str(a * b)
    elif op == 'div':
        if b == 0:
            return "Division by zero", 400
        return str(a / b)
    else:
        return "Invalid operation", 400

@app.route('/echo')
def echo():
    return request.args.get('msg', 'Hello')

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5005)
```

### Backend Start Script

```
nano ~/lab1/start.sh

#!/bin/bash
cd ~/lab1
source venv/bin/activate

MY_IP=$(hostname -I | awk '{print $1}')
echo "Calculator running at http://$MY_IP:5005"

uwsgi --http 0.0.0.0:5005 --wsgi-file app.py --callable app
```
Making Executable ---->>  chmod +x ~/lab1/start.sh

### Run VM-REST ------>>  ./start.sh


## VM-Web

```bash
sudo hostnamectl set-hostname vm-web
exec bash
```
### Install Nginx

```
sudo apt update
sudo apt install nginx -y
systemctl status nginx
```

## Web GUI

```
sudo nano /var/www/html/index.html

<!DOCTYPE html>
<html>
<head>
<title>Cloud Calculator</title>
</head>
<body style="font-family:Arial;padding:40px">

<h2>Cloud Calculator (VM → VM)</h2>

<input type="number" id="a" placeholder="A">
<input type="number" id="b" placeholder="B"><br><br>

<button onclick="calc('add')">+</button>
<button onclick="calc('sub')">-</button>
<button onclick="calc('mul')">*</button>
<button onclick="calc('div')">/</button>

<hr>
Result: <b><span id="result">0</span></b>

<script>
async function calc(op) {
  const a = document.getElementById("a").value;
  const b = document.getElementById("b").value;

  const REST_IP = "127.0.0.1";
  const url = `http://${REST_IP}:5005/calculate?a=${a}&b=${b}&op=${op}`;

  try {
    const r = await fetch(url);
    document.getElementById("result").innerText = await r.text();
  } catch {
    document.getElementById("result").innerText = "Error";
  }
}
</script>

</body>
</html>
```

### Dynamic Script

```
nano ~/start.sh

#!/bin/bash

if [ -z "$1" ]; then
  echo "Usage: start <REST_VM_IP>"
  exit 1
fi

REST_IP=$1
MY_IP=$(hostname -I | awk '{print $1}')

sudo sed -i "s/const REST_IP = \".*\"/const REST_IP = \"$REST_IP\"/" /var/www/html/index.html
sudo systemctl restart nginx

echo "Web UI: http://$MY_IP"
echo "REST API: http://$REST_IP:5005"
```

Making Executable ----- >> chmod +x ~/start.sh      <br>
Run --->>         ./start.sh <REST_IP>       in my case 192.168.122.90


## Aliases

VM-REST --->>   echo "alias start='bash ~/lab1/start.sh'" >> ~/.bashrc  <br>
VM-WEB ---->>   echo "alias start='bash ~/start.sh'" >> ~/.bashrc   <br>

Reload----> source ~/.bashrc


## Learning Outcome

This project demonstrates:
VM networking
Reverse proxy architecture
REST APIs
Microservice communication
Cloud-style service separation


## Common Mistakes in NGINX + Flask VM Calculator Setup

1. Wrong VM IP used
   - Frontend points to incorrect backend IP.
   - Fix: Check VM IP using:
     ip a
   - Update REST VM IP in frontend script.

2. Backend not running
   - Flask/uWSGI server not started.
   - Fix:
     cd ~/lab1
     ./start.sh

3. Port already in use
   - Old process using port 5005.
   - Fix:
     sudo lsof -i :5005
     sudo kill -9 <PID>

4. Firewall blocking traffic
   - Firewall blocks ports 80 or 5005.
   - Fix:
     sudo ufw allow 80
     sudo ufw allow 5005

5. NAT vs Bridge network confusion
   - Host cannot access VM due to network mode.
   - Fix: Ensure VMs are reachable via NAT or bridge network correctly.

6. CORS errors in browser
   - Browser blocks cross-VM requests.
   - Fix:
     pip install flask-cors
   - Enable CORS in Flask app.

7. Virtual environment not activated
   - Flask command not found.
   - Fix:
     source venv/bin/activate

8. Wrong nginx HTML file edited
   - Editing wrong index.html file.
   - Correct location:
     /var/www/html/index.html

9. Nginx not restarted after changes
   - New config not applied.
   - Fix:
     sudo systemctl restart nginx

10. Proxy variables causing install errors
    - Proxy blocks pip/apt downloads.
    - Fix:
      unset http_proxy
      unset https_proxy

