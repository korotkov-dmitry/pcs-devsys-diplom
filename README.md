# Курсовая работа по итогам модуля "DevOps и системное администрирование"

- Процесс установки и настройки ufw
```
vagrant@vagrant:~$ sudo ufw allow 22
Rule added
Rule added (v6)
vagrant@vagrant:~$ sudo ufw allow 443
Rule added
Rule added (v6)
```
- Процесс установки и выпуска сертификата с помощью hashicorp vault
```
vagrant@vagrant:~$ vault server -dev -dev-root-token-id root
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.17.5
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.9.2
             Version Sha: f4c6d873e2767c0d6853b5d9ffc77b0d297bfbdf

==> Vault server started! Log data will stream in below:
...
WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: +vueVhf5ZC3v7fqIha+YnDsu00MVHcY26LgAcZJIVw8=
Root Token: root

Development mode should NOT be used in production installations!
```
В другом терминале:
```
vagrant@vagrant:~$ export VAULT_ADDR='http://127.0.0.1:8200'
vagrant@vagrant:~$ export VAULT_TOKEN=root
vagrant@vagrant:~$ vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.9.2
Storage Type    inmem
Cluster Name    vault-cluster-eff06cca
Cluster ID      88663825-e4d8-551f-a5e8-21022368a813
HA Enabled      false
```
Сертификат
```
vagrant@vagrant:~$ vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/
vagrant@vagrant:~$ vault secrets tune -max-lease-ttl=730h pki
Success! Tuned the secrets engine at: pki/
vagrant@vagrant:~$ vault write -field=certificate pki/root/generate/internal \ common_name="example.com" \ ttl=730h > CA_cert.crt
vagrant@vagrant:~$ vault write pki/config/urls \
>      issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
>      crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
Success! Data written to: pki/config/urls
vagrant@vagrant:~$ vault secrets enable -path=pki_int pki
vagrant@vagrant:~$ vault secrets tune -max-lease-ttl=43800h pki_int
Success! Tuned the secrets engine at: pki_int/
vagrant@vagrant:~$ vault write -format=json pki_int/intermediate/generate/internal \
>      common_name="example.com Intermediate Authority" \
>      | jq -r '.data.csr' > pki_intermediate.csr
vagrant@vagrant:~$ vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr \
>      format=pem_bundle ttl="43800h" \
>      | jq -r '.data.certificate' > intermediate.cert.pem
vagrant@vagrant:~$ vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
Success! Data written to: pki_int/intermediate/set-signed
vagrant@vagrant:~$ vault write pki_int/roles/example-dot-com \
>      allowed_domains="example.com" \
>      allow_subdomains=true \
>      max_ttl="720h"
Success! Data written to: pki_int/roles/example-dot-com
vagrant@vagrant:~$ vault write pki_int/issue/example-dot-com common_name="test.example.com" ttl="24h"
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
expiration          1641719983
issuing_ca          -----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       24:a7:67:6c:25:94:21:d9:ed:1d:8a:52:4f:71:ff:25:57:a9:32:40
```
![cert](https://user-images.githubusercontent.com/92984527/148639881-a002ec76-222a-44ad-be9f-7ab3549d41ec.png)
- Процесс установки и настройки сервера nginx
```
vagrant@vagrant:~$ sudo apt install nginx
vagrant@vagrant:~$ sudo vim /etc/nginx/nginx.conf
...
http {
...
  server {
      listen              443 ssl;
      server_name         test.example.com;
      ssl_certificate     /etc/ssl/example.crt;
      ssl_certificate_key /etc/ssl/example.key;
vagrant@vagrant:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful      
vagrant@vagrant:~$ sudo /etc/init.d/nginx restart
```
![nginx](https://user-images.githubusercontent.com/92984527/149081826-cb8a1b6a-aafa-42fb-aeb7-72afac730f0b.png)
![certificate](https://user-images.githubusercontent.com/92984527/149081895-9e3b895d-8b54-4e45-8822-cbdb49a39b08.png)
- Скрипт генерации нового сертификата работает (сертификат сервера ngnix должен быть "зеленым")
```
vagrant@vagrant:~$ sudo vim create_certificate
vault write -format=json pki_int/issue/example-dot-com common_name="test.example.com" ttl="730h" > /etc/ssl/example.crt

cat /etc/ssl/example.crt | jq -r .data.certificate > /etc/ssl/example.crt.pem
cat /etc/ssl/example.crt | jq -r .data.ca_chain[ ] >> /etc/ssl/example.crt.pem
cat /etc/ssl/example.crt | jq -r .data.private_key > /etc/ssl/example.key
```
- Crontab работает (выберите число и время так, чтобы показать что crontab запускается и делает что надо)
```
vagrant@vagrant:~$ crontab -e
...
* * * * * /home/vagrant/create_certificate
vagrant@vagrant:~$ sudo service rsyslog restart
vagrant@vagrant:~$ sudo service cron restart
vagrant@vagrant:~$ sudo service cron status
● cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2022-01-08 12:16:24 UTC; 3min 31s ago
       Docs: man:cron(8)
   Main PID: 2285 (cron)
      Tasks: 1 (limit: 1071)
     Memory: 1.1M
     CGroup: /system.slice/cron.service
             └─2285 /usr/sbin/cron -f

Jan 08 12:17:02 vagrant CRON[2311]: pam_unix(cron:session): session closed for user vagrant
Jan 08 12:17:02 vagrant CRON[2310]: pam_unix(cron:session): session closed for user root
Jan 08 12:18:01 vagrant CRON[2327]: pam_unix(cron:session): session opened for user vagrant by (uid=0)
Jan 08 12:18:01 vagrant CRON[2328]: (vagrant) CMD (/home/vagrant/create_certificate)
Jan 08 12:18:01 vagrant CRON[2327]: pam_unix(cron:session): session closed for user vagrant
Jan 08 12:19:01 vagrant CRON[2340]: pam_unix(cron:session): session opened for user vagrant by (uid=0)
Jan 08 12:19:01 vagrant CRON[2341]: (vagrant) CMD (/home/vagrant/create_certificate)
Jan 08 12:19:01 vagrant CRON[2340]: pam_unix(cron:session): session closed for user vagrant
```
