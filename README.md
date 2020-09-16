# Scope AR Installation Guide

Provides a way to run the Scope AR server software using Docker in swarm mode.

## Requirements

* Docker swarm cluster with 1, 3 or 3+ nodes
    * Docker version 18.06.0+
    * Each server must have a directory which is shared between all the nodes
* Docker Hub account authorized by Scope AR
* SMTP server for email
* Valid TLS certificate for HTTPS
* MySQL/MariaDB server (optional)
* Cloud storage for user content (optional)
    * S3

## Installation

### Firewall Ports

1. Private network, communication between nodes in the cluster.

    * Swarm cluster

        Port|Protocol|Description
        ---|---|---
        2377|TCP|Swarm cluster management communication
        7946|TCP|Swarm node communication
        7946|UDP|Swarm node communication
        4789|UDP|Swarm overlay network

1. Public network, this could be a WLAN address accessible throughout your organization or a public IP address.

    * Web Stack

        Port|Protocol|Description
        ---|---|---
        80|TCP|HTTP
        443|TCP|HTTPS

    * Comms Stack

        Port|Protocol|Description
        ---|---|---
        24000|UDP|Zeus Service

    * Proxy Stack

        Port|Protocol|Description
        ---|---|---
        8030|TCP|Ping Service
        3478|TCP|Turn Service
        3478|UDP|Turn Service
        49152-65535|UDP|Turn Communication

### Docker Swarm

Each server in the cluster must be running docker in swarm mode. Please see the [swarm mode overview](https://docs.docker.com/engine/swarm/) and [key concepts](https://docs.docker.com/engine/swarm/key-concepts/) for more information.

1. Install docker, please see the [installation instructions](https://docs.docker.com/install/) for your operating system.

1. To verify your installation on each server, you can run the following commands:

    ```bash
    $ docker --version
    $ docker run --rm hello-world
    ```

1. Choose a node to be a manager and sign in to [Docker Hub](https://hub.docker.com/) with the credentials authorized by Scope AR.

    ```bash
    $ docker login
    ```

1. On the same node, initialize the swarm cluster.

    ```bash
    $ docker swarm init --advertise-addr 0.0.0.0:2377
    $ docker swarm join-token manager
    ```

1. For each remaining nodes in the cluster, run the command printed by the previous step. Optional if this is a single node cluster.

    ```bash
    $ docker swarm join --token "<Swarm Token>" "<IP Address>:2377"
    ```

### Shared Directory

* In this guide, the shared directory is referenced as `/mnt/shared`.

* *Single node cluster*: The directory doesn't have to be shared and can exist any where on the file system. If the cluster expands in the future then this directory must be shared.

* *Multiple node cluster*: All nodes in the cluster must have access to a shared directory. It is recommended that this directory is automatically mounted at boot. Options include:

    * [NFS](https://en.wikipedia.org/wiki/Network_File_System) Share
    * Amazon Web Services [EFS](https://aws.amazon.com/efs/)
    * [GlusterFS](https://www.gluster.org/)
    * [Ceph](https://ceph.com/)

1. Create shared directories.

    ```bash
    $ mkdir -p /mnt/shared/cms_private /mnt/shared/cms_public /mnt/shared/db_data /mnt/shared/redis_cms /mnt/shared/redis_zeus /mnt/shared/nginx /mnt/shared/resource_svc
    ```

1. Set file permissions.

    ```bash
    $ chown -R 1000:999 /mnt/shared/cms_private /mnt/shared/cms_public
    $ chown -R 999:999 /mnt/shared/db_data /mnt/shared/redis_cms /mnt/shared/redis_zeus /mnt/shared/nginx /mnt/shared/resource_svc
    $ chmod 600 /mnt/shared
    ```

### Configuration

* In this guide, the configuration files are located in `/opt/scope`
* By default, docker secrets are stored in the directory `/opt/scope/secrets`

#### CMS

File: `/opt/scope/cms.env`

* Secrets (required)

    File|Description
    ---|---
    `secrets/cms_password`|Password for the initial admin user, the password must be at least 8 characters, 1 uppercase letter, 1 lowercase letter, 1 number, and 1 special character
    `secrets/cms_secret_key`|Random string of at least 64 characters: `openssl rand -hex 128`
    `secrets/cms_secret_token`|Random string of at least 64 characters: `openssl rand -hex 128`
    `secrets/cms_smtp_password`|SMTP Password

* Initial administrator user (required)

    Key|Description
    ---|---
    `COMPANY_NAME`|Name of the compay to create
    `FIRST_NAME`|First name of the admin user
    `LAST_NAME`|Last name of the admin user
    `EMAIL`|Email address, used to sign in
    `PHONE_NUMBER`|Phone number of the admin user

* Web site (required)

    Key|Values|Description
    ---|---|---
    `CMS_PROTOCOL`|`https`, `http`|
    `CMS_HOST`||Host name or IP address of single swarm node|Address which users connect to the web site
    `CMS_PORT`|`443`, `80`
    `CMS_AUTH_METHOD`|`db`, `ldap`|Method of authentication, either local database or using LDAP
    `FILE_STORAGE`|`local`, `s3`|Where user content is stored, `local` stores content in the shared directory and `s3` in an AWS [S3](https://aws.amazon.com/s3/) bucket

* Web site (optional)

    Key|Values|Description
    ---|---|---
    `VIRUS_SCAN_ENABLED`|`true`, `false`|Virus scan user uploaded content
    `AUTH_TOKEN_DURATION`|`[count] [unit]`|Authentication tokens are valid for this duration, units can be: `hour`, `hours`, `day`, `days`, `month`, `months`

* SMTP (required)

    Key|Values|Description
    ---|---|---
    `EMAIL_SENDER`||Email address FROM header
    `SMTP_ADDRESS`||Host name of the SMTP server
    `SMTP_PORT`||Port used to connect to the SMTP server
    `SMTP_DOMAIN`||If the SMTP server requires a HELO domain, specify it here (required for Amazon SES)
    `SMTP_USER_NAME`||User name (password is specified above in a docker secret)
    `SMTP_AUTHENTICATION`|`plain`, `login`, `cram_md5`|Type of authentication for the SMTP server (use `login` for Amazon SES)
    `SMTP_OPENSSL_VERIFY_MODE`|`none`, `peer`, `client_once`, `fail_if_no_peer_cert`|Specifies certificate validation when using TLS
    `SMTP_ENABLE_STARTTLS_AUTO`|`true`, `false`|Automatically use TLS if available

* Network Grid

  * Describes how clients connect to the swarm nodes
  * Each server block represents a single swarm node that is located in a geographical region
  * Increase the number in `SERVER_x` for every server

    Key|Default|Description
    ---|---|---
    `SERVER_x_LABEL`|`On Prem`|Label for this group of servers, all servers with the same label are grouped together
    `SERVER_x_ZEUS_EXTERNAL_HOST`|Domain or IP address of the swarm node which clients will use to connect* 
    `SERVER_x_ZEUS_EXTERNAL_PORT`|`24000`|External port
    `SERVER_x_ZEUS_ENCRYPTION_CERT`|Certificate used by Zeus for encryption
    `SERVER_x_PING_EXTERNAL_HOST`|Domain or IP address of the swarm node which clients will use to connect*
    `SERVER_x_PING_EXTERNAL_PORT`|`8030`|External port
    `SERVER_x_TURN_EXTERNAL_HOST`|Domain or IP address of the swarm node which clients will use to connect*
    `SERVER_x_TURN_EXTERNAL_PORT`|`3478`|External port
    `SERVER_x_TUNNEL_EXTERNAL_HOST`|*|Unused
    `SERVER_x_TUNNEL_EXTERNAL_PORT`||Unused

    *Do not use localhost or local IPs (e.g. 127.0.0.1) for the value, instead use externally referenced domain name (fqdn) or external IP of server if required. 

#### Nginx

By default, nginx will start with a self signed certificate for HTTPS requests if none is provided. To use another certificate, either replace the secret file listed below or use a symlink to the actual file.

* Secrets to modify (optional)

    File|Description
    ---|---
    `secrets/nginx_ssl_cert`|TLS/SSL certificate, maps to the nginx setting [ssl_certificate](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_certificate)
    `secrets/nginx_ssl_key`|TLS/SSL certificate secret key, maps to the nginx setting [ssl_certificate_key](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_certificate_key)

#### MariaDB

File: `/opt/scope/mariadb.env`

* If using an external database server, comment out the `mariadb:` section in `web.yml` and configure the required connection details.

* Database configuration (required)

    Key|Description
    ---|---
    `MYSQL_ROOT_USER`|Root user name
    `MYSQL_USER`|User name
    `MYSQL_HOST`|If using the MariaDB docker container, specify `mariadb`, otherwise the host of the database server (such as AWS [RDS](https://aws.amazon.com/rds/))
    `MYSQL_PORT`|Database port
    `MYSQL_DATABASE`|Database name

    File|Description
    ---|---
    `secrets/mysql_root_password`|Root user password
    `secrets/mysql_password`|User password

#### Zeus

Create a new PKCS12 encryption certificate for Zeus. Setting a password on the certificate is optional.

1. Create the certificate.

    ```bash
    $ cd /opt/scope/
    $ openssl req -x509 -newkey rsa:4096 -keyout /tmp/zeus_key.pem -out secrets/zeus_encryption_cert -days 3650 -nodes
    $ openssl pkcs12 -in secrets/zeus_encryption_cert -inkey /tmp/zeus_key.pem -export -out secrets/zeus_encryption_key
    ```

If you created the certificate with a password, update the file `secrets/zeus_encryption_password` with the password.

#### Turn

File: `/opt/scope/turn.env`

* If behind NAT (such as EC2 instances) then one entry of `TURN_EXTERNAL_IPV4_x` is required for every swarm node.

* Turn configuration (required)

    Key|Description
    ---|---
    `TURN_SERVER_PORT`|Listen port for turn
    `TURN_RELAY_PORT_RANGE_MIN`|First port in the relay range
    `TURN_RELAY_PORT_RANGE_MAX`|Last port in the relay range
    `TURN_EXTERNAL_IPV4_x`|Mapping of `<Public IP>/<Private IP>` address for swarm nodes behind NAT (increment `x` for every server)

### LDAP

An LDAP server can be used to manage user authentication. To enable LDAP, configure the following variables in `cms.env`.
When a user authenticates via LDAP for the first time, the CMS will import their name, phone number, and email address from the LDAP directory via the `_ATTRIBUTE` keys.

File: `/opt/scope/cms.env`

* LDAP environment variables (optional)

    Key|Values|Description
    ---|---|---
    `CMS_AUTH_METHOD`|`ldap`|
    `LDAP_HOST`||Host name of the LDAP server
    `LDAP_PORT`|`389`, `636`|Port used to connect to the LDAP server
    `LDAP_SSL`|`true`, `false`|Enable or disable encryption (TLS)
    `LDAP_AUTH_ATTRIBUTE`||LDAP attribute used for authentication (user name)
    `LDAP_AUTH_PASSWORD_TYPE`|`sha`, `md5`|Type of password
    `LDAP_DN_BASE`||LDAP base DN
    `LDAP_ADMIN_USER`||LDAP admin user
    `LDAP_FIRST_NAME_ATTRIBUTE`||LDAP attribute which has the user's first name
    `LDAP_LAST_NAME_ATTRIBUTE`||LDAP attribute which has the user's last name
    `LDAP_EMAIL_ATTRIBUTE`||LDAP attribute which contains the user's email address
    `LDAP_PHONE_NUMBER_ATTRIBUTE`||LDAP attribute which contains the user's phone number
    `LDAP_CREATE_USER`||Set to false by default. When set to true, a user trying to login, that currently does not exist in LDAP, is created in LDAP automatically. 

* LDAP secrets (optional)

    File|Description
    ---|---
    `secrets/cms_ldap_admin_password`|LDAP admin password

**Note:**

* When LDAP is enabled on existing installations, users will only be able to continue to use Scope AR products if:

    * their LDAP account email address exactly matches the email address already entered in the CMS
    * their LDAP account can be authenticated via the configuration provided

    It is likely that any existing users will need to be notified that their password may have changed.

    No data will be removed or deleted for any existing users who would not able to authenticate via LDAP.

* If your LDAP server exists outside of your Docker Swarm, you may need to add a host name mapping to `web.yml`. This will allow containers running within Swarm to resolve addresses from outside of Docker's internal DNS:

    ```
    services:
      cms:
        ...
        extra_hosts:
        - "example.com:192.168.1.123"
    ```

### AWS

Enables users content to be stored in a S3 bucket.   Both `AWS_ACCESS_KEY_ID` and

File: `/opt/scope/cms.env`

* AWS configuration (optional)

    Key|Description
    ---|---
    `FILE_STORAGE`|Set to `s3`
    `CMS_DEFAULT_S3_BUCKET`|The S3 bucket to store user content when `FILE_STORAGE` is `s3`
    `CMS_DEFAULT_S3_REGION`|The region of the S3 bucket specified by `CMS_DEFAULT_S3_BUCKET`

**Note:**

* Instead of setting `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in the environment, consider creating an IAM role and attaching the role to each swarm node. To give the nodes access to the S3 bucket, attach the following policy to the role (replacing `<s3 bucket>` with the name of the S3 bucket):

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "",
                "Effect": "Allow",
                "Action": "s3:ListBucket",
                "Resource": "arn:aws:s3:::<s3 bucket>
            },
            {
                "Sid": "",
                "Effect": "Allow",
                "Action": [
                    "s3:PutObjectAcl",
                    "s3:PutObject",
                    "s3:GetObject"
                ],
                "Resource": "arn:aws:s3:::<s3 bucket>/*"
            }
        ]
    }
    ```

## Deployment

1. Change directory.

    ```bash
    $ cd /opt/scope/
    ```

1. If any nodes in the cluster do not have access to the internet, please see the section **Installation Without the Internet**.

1. Deploy the stacks. Depending on the speed of the machines, internet and other factors, initialization may take some time.

    ```bash
    # If the nodes do no have access to the internet, add the command line option: --resolve-image=never
    $ docker stack deploy --compose-file web.yml --prune --with-registry-auth web
    $ docker stack deploy --compose-file comms.yml --prune --with-registry-auth comms
    $ docker stack deploy --compose-file proxy.yml --prune --with-registry-auth proxy
    ```

1. To monitor migration progress, the list of services can be viewed. Some services can take several minutes to start and will show as `0` until their health check has passed.

    ```bash
    $ docker service ls
    ```

### Common Docker Swarm Commands

* `docker service ls` - List running services. Service names can be found `NAME` column for later commands.
* `docker service ps --no-trunc <service name>` - Status of a running service.
* `docker service logs --since 1h -f <service name>` - Tail logs for a service.
* `docker stack rm <stack name>` - Remove a stack, stack names are: `web`, `comms`, `proxy`. Removing a stack can take some time (repeat the command until nothing is found in the stack).

## Troubleshooting

1. Check all ports which are listed under **Firewall Ports** in `README.md` are open.

1. If the shared directory is a network share or remote, check that it is mounted.

1. Check status of all services. Service name can be found under the `NAME` column for subsequent commands.

    ```bash
    $ docker service ls
    ```

1. If a container cannot start, check any error messages for the service.

    ```bash
    $ docker service ps --no-trunc web_cms
    ```

1. View the logs for a service.

    ```
    $ docker service logs --since 1h -f web_cms
    ```

### Installation Without the Internet

If any nodes in the cluster do not have access to the internet, then the docker images must first be downloaded and then transferred manually to the node.

The required images can be downloaded on a machine with `docker` and a connection to the internet. Docker can be [installed](https://docs.docker.com/install/) on Linux, Windows and MacOS.

1. Sign in to [Docker Hub](https://hub.docker.com/) with the credentials authorized by Scope AR.

    ```bash
    $ docker login
    ```

1. Pull images.

    ```bash
    $ docker pull scopear/scope-cms:on-prem-2.13.2
    $ docker pull scopear/docker-nginx:on-prem-2.13.2
    $ docker pull scopear/docker-db:on-prem-2.13.2
    $ docker pull scopear/docker-zeus:on-prem-2.13.2
    $ docker pull scopear/docker-redis:on-prem-2.13.2
    $ docker pull scopear/docker-turnserver:on-prem-2.13.2
    ```

1. Export images.

    ```bash
    $ docker save -o scope-cms.tar         scopear/scope-cms:on-prem-2.13.2
    $ docker save -o docker-nginx.tar      scopear/docker-nginx:on-prem-2.13.2
    $ docker save -o docker-db.tar         scopear/docker-db:on-prem-2.13.2
    $ docker save -o docker-zeus.tar       scopear/docker-zeus:on-prem-2.13.2
    $ docker save -o docker-redis.tar      scopear/docker-redis:on-prem-2.13.2
    $ docker save -o docker-turnserver.tar scopear/docker-turnserver:on-prem-2.13.2
    ```

1. Docker images will saved to the respective `.tar` files. Once all the `.tar` files have been transferred to each node in the cluster, change directory to where the files are located and run:

    ```bash
    $ # On the target server
    $ docker load -i scope-cms.tar
    $ docker load -i docker-nginx.tar
    $ docker load -i docker-db.tar
    $ docker load -i docker-zeus.tar
    $ docker load -i docker-redis.tar
    $ docker load -i docker-turnserver.tar
    ```

## Verifying a Proper Setup

After installing or updating Scope AR Intranet there are a few ways to verify that everything is working as expected.

### Verifying the CMS

1. Using a device that can run Remote AR and WorkLink, open the browser and try to access the CMS using your configured domain/IP.
1. Once loaded, log in using the login credentials configured in the `cms.env` file.
    * It's best to change your password to something not mentioned in the `cms.env` file for security reasons.

### Verifying Remote AR

1. Install and/or launch Remote AR.
1. Log out if you are currently logged in.
1. From the login screen select the "Advanced Settings" button.
1. Enter your configured domain/IP and press "Save".
1. Restart the app after setting those values.
1. Once launched, login using your CMS credentials.
1. If the login is successful, the app is communicating with the CMS.
1. After logging in, Tap on "AR view" on the bottom of contact list.
1. You will see a warning message asking if you would like to return to contact list. Press "no".
1. Tap on settings gear icon on the right side.
1. Under Diagnostics, Every status light should be green.
1. If every light is green, use a second device configured with Remote AR and try to make a connection.

### Verifying WorkLink Player

1. Install and/or launch WorkLink Player app.
1. Log out if you are currently logged in.
1. From the login screen, select the "settings" gear icon from top right.
1. A menu will appear, select "Custom CMS Endpoint".
1. Enter your configured Domain/IP. Restart the app after setting those values. (ignore "Custom Guest Authentication" field, leave it blank)
1. Once relaunched login using your CMS credentials.
1. If logging in works, then WorkLink is setup properly.

### Verifying WorkLink Launcher

1. Install and/or launch WorkLink Launcher.
1. From the top menu bar, select "WorkLink" > "Custom CMS Endpoint"
1. Enter your configured domain/IP and press "OK". Restart the app after setting those values.
1. Once relaunched, login using your CMS credentials.
1. If logging in works, then WorkLink Launcher is setup properly.

### Verifying WorkLink Authoring Platform

1. Create a new WorkLink project or Launch an existing WorkLink project (Refer to the WorkLink Authoring Platform documentation for more details if needed).
1. Once launched, click on WorkLink from top menu bar > Options > Custom CMS Endpoint.
1. Select "Custom" from CMS API Mode drop-down.
1. Enter your configured domain/IP in "CMS Custom API URL" field.
1. Click on "Test Connection" to check connectivity with CMS.
1. Click on "Save Changes" and close this window.
1. From top menu bar, go to the "WorkLink" > "Publish" and login using your CMS credentials.
1. You are now set up to publish WorkLink Projects on your CMS instance.
