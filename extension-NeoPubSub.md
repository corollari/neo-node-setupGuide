## Install & Set up redis
<!-- See https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-redis-on-ubuntu-18-04 -->
```bash
sudo apt update
sudo apt install -y redis-server
```

Then edit the following file:
```
sudo vi /etc/redis/redis.conf
```
and change the line 
```
supervised no
```
to
```
supervised systemd
```

then restart redis:
```
sudo systemctl restart redis.service
```

## Install NeoPubSub plugin
Follow the instructions in the `Installation` section of smartbnb/NeoPubSub/README.md to compile the plugin and install it.

## Install node
```
sudo apt-get install curl
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt-get install -y nodejs
```

## Set up redis2ws
```
git clone https://github.com/safudex/smartbnb.git
cd smartbnb/NeoPubSub/redis2ws
npm install
```

## Set up a supervisor for redis2ws
Open:
```bash
sudo vi /etc/systemd/system/redis2ws.service
```

And paste this code:
```
[Unit]
After=network-online.target
Requires=network-online.target
[Service]
WorkingDirectory=/home/ubuntu/smartbnb/NeoPubSub/redis2ws
ExecStart=/usr/bin/node index.js
ExecStop=/bin/kill -SIGINT `ps ax | index.js | grep -v grep | awk ‘{print $1}’`
Restart=always
StandardInput=tty-force
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=redis2ws
User=ubuntu
Group=ubuntu
[Install]
WantedBy=multi-user.target
```

Then activate it:
```
sudo systemctl enable redis2ws
sudo systemctl start redis2ws
```

## Set up nginx websocket proxy
Open the following file replacing example.com with your domain:
```
sudo vi /etc/nginx/sites-enabled/example.com.conf
```

And inside the first `server{}` instance add the following lines:
```
location /ws/ {
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_pass "http://localhost:8000/";
}
```

Then restart nginx:
```
sudo systemctl restart nginx
```
