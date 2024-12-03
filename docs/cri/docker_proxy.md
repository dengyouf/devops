# Docker代理

```shell
mkdir -p /etc/systemd/system/docker.service.d
cat > /etc/systemd/system/docker.service.d/proxy.conf << EOF
[Service]
Environment="HTTP_PROXY=http://172.16.192.1:7890/"
Environment="HTTPS_PROXY=http://172.16.192.1:7890/"
Environment="NO_PROXY=localhost,127.0.0.1,.linux.io,172.17.0.0/16"
EOF

systemctl daemon-reload && systemctl restart docker

docker pull registry.k8s.io/pause:3.6
```