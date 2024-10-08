# Overview
Gitea is a painless self-hosted all-in-one software development service, including Git hosting, code review, team collaboration, and package registry. It's open source under MIT license. It is designed to be lightweight, easy to use, and highly customizable, making it an ideal choice for both small teams and large organizations

Gitea features an integrated CI/CD system, Gitea Actions, that is compatible with GitHub Actions. Users can create workflows using the familiar YAML format or utilize over 20K existing plugins.

# Why Choose Gitea

- Open Source: Gitea is open-source software, meaning you have full control over your repositories and data.
- Self-Hosted: You can host Gitea on your own infrastructure, giving you complete autonomy over your Git service.
- Community Support: Gitea has a vibrant community of developers and users who contribute to its continuous improvement.
- Scalable: Whether you're a small startup or a large enterprise, Gitea can grow with your needs.
- Simplified Maintenance: Gitea's user-friendly interface and straightforward setup reduce the need for extensive IT support.

# System Requirements
| Name | 		Description | information |
|-- |-- |-- |
| Database | postgresql | Host in server or rds |
| Authentication | google oauth | GCP console Oauth |
| Domain	| server-gitea.com | DNS aliyun

# Instalation
Gitea dashboard will running in docker swarm server.
## Instalation & Configuration Steps
#### 1. Deploy Gitea in swarm devops central 

```
- Using docker context to deploy from local-remote, then deploy using stack deployment.
```
docker stack deploy -c gitea.yml gitea
```
- or it could be deployed from portainer dashboard

### 2. Gitea Initial Configuration
- Acces gitea dashbaord with http://ip_server:7500 for initial configuration
- in this configuration, we need to make sure that database credential connection settings is correct. Then create administrator user account settins to input username and password.
- then click Install Gitea

### 3 Domain Configuration and Google Oauth Authentication 
- for domain of gitea dashboard, is exposed to server vm-be. so domain setting is configure in server vm-be using nginx reverse proxy to expose endpoint in server devops.
```
server {
  listen 80;
  server_name server_name;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;

  ssl_prefer_server_ciphers on;
  ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
  ssl_ecdh_curve secp384r1;
  ssl_session_cache shared:SSL:10m;
  ssl_session_tickets off;
  ssl_stapling on;
  ssl_stapling_verify on;

  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

  root /var/www/html/;


  ssl_certificate /etc/nginx/ssl/domain.pem;
  ssl_certificate_key /etc/nginx/ssl/domain.key;

  server_name server_name;
  underscores_in_headers on;
  client_max_body_size 16M;

  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_set_header x_consid $http_x_consid;
    proxy_set_header x_timestamp $http_x_timestamp;
    proxy_set_header x_signature $http_x_signature;
    proxy_set_header x_authorization $http_x_authorization;

    proxy_pass http://ip_server.nip.io:7500/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_cache_bypass $http_upgrade;
  }

}

```
- For authentication, gitea uses google oauth. so gitea domain needs to be registed google console oauth credential
- enable google-auth by editing config file grafana.ini . then add configuration below. make sure to fill client_id and client_secret of google-aouth
```
[auth.google]
enabled = true
allow_sign_up = true
auto_login = false
client_id = ****
client_secret = ****
scopes = https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/userinfo.email
auth_url = https://accounts.google.com/o/oauth2/auth
token_url = https://accounts.google.com/o/oauth2/token
api_url = https://www.googleapis.com/oauth2/v1/userinfo
allowed_domains = google.com
hosted_domain = google.com
use_pkce = true
```
- then save configuration file. restart service gitea, then akses gitea dashboard and login using your fit.email

### 4. Configure Act-Runner as Runner Of Gitea
- ssh to server 
- download gitea-act-runner package here https://dl.gitea.com/act_runner/ we use version: act_runner-0.2.10-linux-amd64
- When download the binary, make sure that permission is allowed```
```
chmod +x act_runner
```
-  generate a configuration file by running the following command:
```
./act_runner generate-config
```
- then edit config.yaml to run jobs in the host directly, you can change it to ubuntu-22.04:host
```
labels:
     - "ubuntu-latest:host"
```
- create file script.sh 
```
#!/bin/bash

./act_runner daemon --config config.yaml
```
- then run script below to make runner run in background
```
nohup ./script.sh &
```
- act runner already configure
