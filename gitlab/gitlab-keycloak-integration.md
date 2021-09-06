# Goal
Install a **CE edition self-managed Gitlab** which can **work with datalad (git lfs/git annex)** and also need to **change the relative url of the Gitlab** to make the final url like: `example.com/gitlab`

# Conclusion
We have tested Gitlab V11.7.4 and V14.1.3, and **only V11.7.4 can work with datalad (git lfs/gitl annex)**. Also, in order to make datalad work properly, **Gitlab website has to have valid https certificate**. The test was done on September 2021.

# Package (V11.7.4)

1. Login to the server you are going install the Gitlab package and **download the package** to local by running below command:
```
wget --content-disposition wget --content-disposition https://packages.gitlab.com/gitlab/gitlab-ce/packages/ubuntu/xenial/gitlab-ce_11.7.4-ce.0_amd64.deb/download.deb
```
 

2. **Install** the package downloaded above:
```
sudo dpkg -i gitlab-ce_11.7.4-ce.0_amd64.deb
```
 

3. Modify the configuration by **changing the relative url** and **integrating with Keycloak** :
>Note: Make sure to add the relative path /git in the end  
>Note: Please replace the value in <> accordingly
```
external_url 'https://<your gitlab domain name>/git'
gitlab_rails['time_zone'] = 'America/Toronto'
gitlab_rails['registry_enabled'] = false
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"
gitlab_rails['backup_upload_connection'] = {
        :provider => 'Local',
        :local_root => '/var/opt/gitlab'
}
gitlab_rails['backup_keep_time'] = 1814400
gitlab_rails['backup_upload_remote_directory'] = '.'
gitlab_rails['lfs_enabled'] = true
gitlab_rails['lfs_storage_path'] = "/var/opt/gitlab/gitlab-rails/shared/lfs-objects"
gitlab_rails['gitlab_email_from'] = "gitlab@example.com"
prometheus['enable'] = false
gitlab_rails['registry_enabled'] = false
postgresql['shared_buffers'] = "1024MB"
nginx['redirect_http_to_https'] = true
nginx['listen_port'] = 80
nginx['listen_https'] = false
gitlab_rails['initial_root_password'] = '<enter your initial password>'
gitlab_rails['omniauth_enabled'] = true
gitlab_rails['omniauth_allow_single_sign_on'] = ['saml']
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_auto_link_saml_user'] = true
gitlab_rails['omniauth_providers'] = [
{
  name: 'saml',
  args: {
    assertion_consumer_service_url: 'https://<your gitlab domain name>/git/users/auth/saml/callback',
    # This certificate can be fetched at Keycloak: "Realm Settings" -> "Keys" -> "RS256" -> "Certificate"
    idp_cert: " -----BEGIN CERTIFICATE-----
\n <your keycloak saml certificate> \n",
    idp_sso_target_url: 'https://<your keycloak domain name>/auth/realms/<your realm name>/protocol/saml/clients/gitlab-saml',
     issuer: '<keycloak client>',
     name_identifier_format: 'urn:oasis:names:tc:SAML:2.0:nameid-format:persistent'
 },
  label: 'Keycloak Login',
  attribute_statements: {
    first_name: 'first_name',
    last_name: 'last_name',
    name: 'name',
    username: 'name',
    email: 'email'
  },
}
]
```
 

4. **Reconfigure** the gitlab
```
sudo gitlab-ctl reconfigure
```

5. Go to **Keycloak** and **create a client** called `gitlab-saml`, and then add below mappers:
```
    Name: name
        Mapper Type: User Property
        Property: Username
        Friendly Name: Username
        SAML Attribute Name: name
        SAML Attribute NameFormat: Basic
        
    Name: email
        Mapper Type: User Property
        Property: Email
        Friendly Name: Email
        SAML Attribute Name: email
        SAML Attribute NameFormat: Basic
        
    Name: first_name
        Mapper Type: User Property
        Property: FirstName
        Friendly Name: First Name
        SAML Attribute Name: first_name
        SAML Attribute NameFormat: Basic
        
    Name: last_name
        Mapper Type: User Property
        Property: LastName
        Friendly Name: Last Name
        SAML Attribute Name: last_name
        SAML Attribute NameFormat: Basic
        
    Name: roles
        Mapper Type: Role list
        Role attribute name: roles
        Friendly Name: Roles
        SAML Attribute NameFormat: Basic
        Single Role Attribute: On
```
>Note: All of the mappers have “Consent Required” set to Off.

 

6. If you would like to **re-direct the traffic from kubernetes' ingress**, you can deploy an ingress as below:
```
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gitlab-external-ingress
  namespace: gitlab
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/add-base-url: 'true'
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: "25m"
spec:
  rules:
  - host: 
    http:
      paths:
      - path: /git(/|$).*
        backend:
          serviceName: gitlab-external
          servicePort: 80
```
 

7. Deploy a **service** & **endpoint** in kubernetes cluster to redirect traffic the external gitlab  
```
apiVersion: v1
kind: Service
metadata:
  name: gitlab-external
  namespace: gitlab
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: v1
kind: Endpoints
metadata:
  name: gitlab-external
  namespace: gitlab
subsets:
  - addresses:
      - ip: <your gitlab internal ip address>
    ports:
      - port: 80
```

# Datalad Verification Steps
1. Go to the Gitlab website and fetch the **access token** via **“Settings”** (On the top right dropdown menu) → **“Access Token”** (On the left navigation lane) → Type the name of the token and select **“api”**, and then click **“Create personal access token”**

2. Add **python config** with url and access token you got from Step 1
```
$ vim ~/.python-gitlab.cfg

[gitlab.site]
url = https://<your gitlab domain name>/gitlab
ssl_verify = True
private_token = <paste your access token here>
```
 

3. **Create dataset**
```
cd /tmp && datalad create newrepo && cd newrepo
```
 

4. **Set git config**
```
git config  http.sslVerify false
git config annex.security.allowed-ip-addresses all
```
 

5. **Copy and save** file to dataset
>Please see the sample_file.rds file in below attachment

```
cp /path/to/sample_file.rds .
datalad save -m "added file"
```

 

6. **Create repo** on gitlab
```
datalad create-sibling-gitlab -d . --site gitlab.site --project datalad_testrepo
```
 

7. Set up **push remote**
```
git annex initremote lfs-remote type=git-lfs encryption=none url=https://<your gitlab domain name>/root/datalad_testrepo
```
 

8. **Push** to gitlab
```
git annex copy --to=lfs-remote .
```

>If you see below return, it means it works. Otherwise, it will response with some error.

```
$ git annex copy --to=lfs-remote .
copy sample_file.rds (to lfs-remote...) 
ok                                   
(recording state in git...) 
```
 
# Gitlab Command
## Regular Operation
```
sudo gitlab-ctl status
sudo gitlab-ctl start
sudo gitlab-ctl stop
sudo gitlab-ctl restart
```

## Check logs
```
sudo gitlab-ctl tail
```

# Troubleshooting
```
sudo gitlab-rake gitlab:check --trace
sudo gitlab-rake db:migrate:status --trace
sudo gitlab-rake db:migrate
```

## Reset User Password
```
sudo gitlab-rake "gitlab:password:reset[root]"
sudo gitlab-rails console
user = User.where(id: 1).first
user = User.find_by_username 'root'
user.password = 'indoc2021!'
user.password_confirmation = 'indoc2021!'
user.send_only_admin_changed_your_password_notification!
user.save!
Datalad (git lfs/git annex) Testing Procedure
```


```
