# Buzz on Ubuntu

> https://github.com/block/buzz

## git

```
git init proxmox-homelab
cd proxmox-homelab
git branch -m master main
git remote add -f origin https://github.com/cutefly/proxmox-homelab.git

echo "/agents/buzz" >> .git/info/sparse-checkout
git pull origin main
```

## installation

```
cd agents/buzz
# .env 설정
docker compose up -d

# member 추가
# member 추가
$ docker exec -it buzz-prod-relay-1 buzz-admin add-member --pubkey $CLENT_PUBKEY
```

## Resource
- Memory : 4G
- Swap : 2G
- Cores : 4
- Root Disk : 16G

## Network
- IP Addresses : 192.168.219.127/24

## DNS
- Hostname : ubunut-buzz
