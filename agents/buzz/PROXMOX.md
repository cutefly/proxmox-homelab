# Buzz on Ubuntu

> https://github.com/block/buzz

```
ubuntu 26.04
zsh
oh-my-zsh
docker
tmux
```

## installation

> https://github.com/block/buzz

```sh
git clone https://github.com/block/buzz.git && cd buzz/deploy/compose

# minio 이미지 인증 오류 대응(compose.yml)
# https://github.com/block/buzz/pkgs/container/buzz-minio
minio 이미지를 ghcr.io/block/buzz-minio:latest 로 교체
minio/mc 이미지를 quay.io/minio/aistor/mc:RELEASE.2025-08-29T21-30-41Z 로 교체

docker compose up -d
```

## 직접 빌드 & relay 실행

```sh
git clone https://github.com/block/buzz.git && cd buzz

# minio 이미지 인증 오류 대응(docker-compose.yml)
# https://github.com/block/buzz/pkgs/container/buzz-minio
minio 이미지를 ghcr.io/block/buzz-minio:latest 로 교체
minio/mc 이미지를 quay.io/minio/aistor/mc:RELEASE.2025-08-29T21-30-41Z 로 교체

. ./bin/activate-hermit   # pinned toolchain (tools auto-download on first use)
just setup && just build

just relay
```

## server

```sh
$ docker run --rm --entrypoint /usr/local/bin/buzz-admin ghcr.io/block/buzz:main generate-key
Public key:  e64b47f99fe8331318105d2b59fe87b10f2f86c66b0c132c361368a96d9041a1
Secret key:  e274196a0c4230f5e2d2d4e0cbdc0ac3d6a0720082caf50939378d977458f85e
RELAY_OWNER_PUBKEY=e64b47f99fe8331318105d2b59fe87b10f2f86c66b0c132c361368a96d9041a1
BUZZ_RELAY_PRIVATE_KEY=e274196a0c4230f5e2d2d4e0cbdc0ac3d6a0720082caf50939378d977458f85e

# BUZZ_GIT_HOOK_HMAC_SECRET 생성
$ openssl rand -hex 32
BUZZ_GIT_HOOK_HMAC_SECRET=207769396a3c567ac4801b41f0e1e478ce283331594fdadd46fc50263de46501

# member 추가
$ docker exec -it buzz-prod-relay-1 buzz-admin add-member --pubkey npub1lpan3yzczqc3ldpye5k47cm0acujd4st3v3qaaqufulpfcjcpr3sm9yqrj

```

## desktop

### m1 macbook air

```text
# private identity key
nsec1ypcd94axzcv7jk7a95k6s9ve9ltz9cpdntgp6yehwnwewvdepwxs737h9l

# public id
npub1lpan3yzczqc3ldpye5k47cm0acujd4st3v3qaaqufulpfcjcpr3sm9yqrj
```

### m1 macbook pro

```text
# private identity key
nsec1mezsa3c4j0s7lxqrrgeun0el28jx4j5cm4eltd2jcrj5m3k0kqfqchuwu3

# public id
npub19vqej5pzz0mlmmd482shyhqc3pcjhvaaeyzmfkvlu2yml2uerassqd65ge
```
