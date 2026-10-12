# Nexus

## 배포 준비

```
git init dockhand-stacks
cd dockhand-stacks
git branch -m master main
git config core.sparseCheckout true
git remote add -f origin https://github.com/cutefly/dockhand-stacks.git
echo "/devops/nexus" > .git/info/sparse-checkout
git pull origin main
```
## Single sign on with keycloak

> [https://muhammad-irfan-tahir.medium.com/part-1-keycloak-sso-with-jenkins-oidc-authentication-and-2fa-enforcement-b94096a82387](https://support.sonatype.com/hc/en-us/articles/47814909115539-Integrating-Keycloak-with-Sonatype-Platform-using-OpenID-Connect-OIDC)

```
OIDC, SAML 인증은 Pro 버전을 이용해야 가능함.(3.86 이후로는 Community Edition에서 사용 불가)
```
