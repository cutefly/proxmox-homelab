# Gitlab

## 배포 준비

```
git init dockhand-stacks
cd dockhand-stacks
git branch -m master main
git config core.sparseCheckout true
git remote add -f origin https://github.com/cutefly/dockhand-stacks.git
echo "/devops/gitlab" > .git/info/sparse-checkout
git pull origin main
```

## Single sign on with keycloak

> https://share.gemini.google/8swjBH2V3ryx

```
Gitlab에서
Admin Area (스패너 아이콘) ➔ Settings ➔ General
Sign-in restrictions
After sign-out path
https://keycloak.club012.com/realms/devops/protocol/openid-connect/logout?client_id=gitlab&post_logout_redirect_uri=https%3A%2F%2Fgitlab.club012.com%2Fusers%2Fsign_in

keycloak에서
Keycloak Admin Console ➔ Clients ➔ 생성한 gitlab 클라이언트
Valid post logout redirect URIs
https://gitlab.club012.com/*
```

## upgrade path

```
16.11.5
16.11.10
17.1.8
17.3.7
17.5.5
17.8.7
17.11.7
18.2.8
18.5.7
18.8.11
18.11.11
# update 2026.10.12
18.11.12
19.2.7
19.3.3
```
