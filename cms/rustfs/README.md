# RustFS high-performance, distributed object storage system

## 배포 준비

```
git init proxmox-homelab
cd proxmox-homelab
git branch -m master main
git config core.sparseCheckout true
git remote add -f origin https://github.com/cutefly/proxmox-homelab.git
echo "cms/rustfs" > .git/info/sparse-checkout
git pull origin main
```

https://github.com/rustfs/rustfs

## Docker quick start

> The RustFS container runs as a non-root user rustfs (UID/GID 10001:10001). If you bind-mount host directories with Docker or Compose, every mounted path must be writable by that user, otherwise startup may fail with permission denied errors. This applies to data directories, log directories, and TLS certificate directories when RUSTFS_TLS_PATH is enabled.

```
# Create data and logs directories
mkdir -p data logs config

# Change the owner of these directories
sudo chown -R 10001:10001 data logs config

# Using latest version
docker run -d -p 9000:9000 -p 9001:9001 -v $(pwd)/data:/data -v $(pwd)/logs:/logs rustfs/rustfs:latest

# Using specific version
docker run -d -p 9000:9000 -p 9001:9001 -v $(pwd)/data:/data -v $(pwd)/logs:/logs rustfs/rustfs:1.0.0-rc.1
```

## Rust cli

```sh
curl -LO https://github.com/rustfs/cli/releases/download/v0.1.31/rustfs-cli-linux-amd64-v0.1.31.tar.gz
tar -xf rustfs-cli-linux-amd64-v0.1.31.tar.gz
sudo mv rc /usr/local/bin/

rc alias set minio1 https://s3.club012.com admin '${RUSTFS_ROOT_PASSWORD}'
rc alias set minio2 https://s3-replica.club012.com admin '${RUSTFS_ROOT_PASSWORD}'
rc alias list


rc admin service-account create minio1 s4miqJiRVhEqRb0fdKRF Ivx7bTD94vlZaDpQXJtSlfO2KlJqcZvbFHPMcodG

```

## encryption

### SSE-S3

```
# bucket 생성
rc bucket create --with-lock minio1/kyc-images

# lifecycle 설정
rc bucket lifecycle rule import minio1/kyc-images policy/ilm-policy.json
rc bucket lifecycle rule export minio1/kyc-images
rc bucket lifecycle rule list minio1/kyc-images

# bucket 암호화 설정
rc bucket encryption set minio1/kyc-images --mode sse-s3
# bucket 암호화 정보
rc bucket encryption info minio1/kyc-images
Bucket: kyc-images
Encryption: Configured
Mode: SSE-S3
# bucket 암호화 해제
rc bucket encryption clear minio1/kyc-images

# replication 설정
rc admin replicate add minio1 minio2
rc admin replicate status minio1
rc admin replicate info minio1
```

### SSE-KMS

```
rc admin kms status minio1
rc admin kms key create minio1 --name rustfs-default-key
rc admin kms key status minio1 rustfs-default-key

# bucket 생성
rc bucket create --with-lock minio1/vault-sse-kms

# lifecycle 설정(옵션)
rc bucket lifecycle rule import minio1/vault-sse-kms policy/ilm-policy.json
rc bucket lifecycle rule export minio1/vault-sse-kms
rc bucket lifecycle rule list minio1/vault-sse-kms

# bucket 암호화 설정
rc bucket encryption set minio1/vault-sse-kms --mode sse-kms
# key id 옵션 (RUSTFS_KMS_DEFAULT_KEY_ID=rustfs-default-key)
rc bucket encryption set minio1/vault-sse-kms --mode sse-kms --key-id rustfs-default-key

# bucket 암호화 정보
rc bucket encryption info minio1/vault-sse-kms
Bucket: vault-sse-kms
Encryption: Configured
Mode: SSE-KMS
KMS Key ID: rustfs-default-key
# bucket 암호화 해제
rc bucket encryption clear minio1/vault-sse-kms
```

## Keycloak을 이용한 OIDC 연동

```
keycloak에 rustfs client를 생성
Valid redirect URIs
- https://rustfs.club012.com/rustfs/admin/v3/oidc/callback/default
- https://rustfs.club012.com/rustfs/console/auth/login
Client scopes에 groups 추가(rustfs-dedicated도 가능)
```

### 권한매핑

```
keycloak에 groups나 policy claim을 이용하여 rustfs의 policy와 매핑하는 방법이 있으나
group이나 policy의 값이 rustfs의 policy에 정의되어 있지 않으면 오류 발생
환경변수 설정 시 Keycloak 매퍼의 Token Claim Name과 일치
RUSTFS_IDENTITY_OPENID_CLAIM_NAME=groups
RUSTFS_IDENTITY_OPENID_CLAIM_NAME=policy # policy attribute mapper를 추가 필요

rustfs에 group을 추가하고 policy를 설정하고, keycloak의 groups과 매핑하는 방법이 효율적임.
(별도의 속성을 추가하지 않아도 무방)
사전에 rustfs에 group을 추가하고 정책을 할당함.
예) jenkins-admin 그룹에 consoleAdmin 권한 할당
```
