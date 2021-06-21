---
layout: post
title:  "Federation and Identity Brokering on OpenStack using OpenID and Keycloak + FreeIPA"
date:   2021-06-20 14.30:00 +0700
categories: linux
author: xb4dc0d3
tags: Cloud,Sysadmin,Auth
---

## Prerequisites
1. FreeIPA and OpenStack is already configured. OpenStack is configured using [Kolla Ansible](https://docs.openstack.org/kolla-ansible/latest/).
2. Keycloak is already configured and federated with FreeIPA (directory service).
3. In this example the current IP and port that has been used
   ```bash
   Keycloak: http://10.20.20.14:8081
   OpenStack: http://10.20.20.100:5000
   ```

## Setup OpenID configuration on Keycloak
<b>This operation is performed on the Keycloak Dashboard.</b>

1. Go to the <b>realm</b> that previously created, for example "freeipa-realm", select `Configure > Clients > Create`, then fill the following section
    ```bash
    Client ID: openstack-client 
    Client Protocol: openid-connect
    ```

2. Adjust the authentication flow using OpenID.

    ```bash
    Client ID: openstack-client 
    Client Protocol: openid-connect
    Access Type: Confidential # define the username/password mechanism
    Standard Flow Enabled: On
    Implicit Flow Enabled: On
    Direct Access Grants Enabled: On

    Valid Redirect URIs *: https://10.20.20.100:5000/v3/OS-FEDERATION/identity_providers/openidtest/protocols/openid/auth 
    Backchannel Logout Session Required: On
    ```
    
    <p style="color:orange">* Note: The valid redirect URIs must match the endpoint on OpenStack Keystone </p>

3. On `Clients > Credentials` tab, you will see the Client ID and Secret.
   
    ```bash
    Client Authenticator: Client ID
    Secret: XXXXXXXXXXXXX
    ```

## Configure OpenID on Keystone and Horizon 

<b>This operation is performed on the OpenStack host.</b>

1. Create directories for Keystone and Horizon services. This directory is used to put the configured files. 

    ```bash
    $ sudo mkdir /etc/kolla/config/horizon
    $ sudo mkdir /etc/kolla/config/keystone
    ```

2. Create the file and change the owner.
    ```bash
    $ sudo touch /etc/kolla/config/horizon/local_settings
    $ sudo touch /etc/kolla/config/keystone/keystone.conf
    $ sudo touch /etc/kolla/config/keystone/wsgi-keystone.conf

    $ sudo chown -R $USER:$USER /etc/kolla/config/horizon/
    $ sudo chown -R $USER:$USER /etc/kolla/config/keystone/
    ```

### Configure federating resource on Keystone service
1. Create identity provider object on Keystone

    Remote id value is an issuer on OpenID. To see the issuer you can obtained from the OpenID configuration.
    By accessing the Keycloak `.well-known/openid-configuration` endpoint.

    For example: <br>
    `http://10.20.20.14:8081/auth/realms/freeipa-realm/.well-known/openid-configuration`

    Then, create the object by running the command below
    ```bash
    $ openstack identity provider set --remote-id http://10.20.20.14:8081/auth/realms/freeipa-realm openidtest
    ```

3. Create a mapping rules for remote user attribute. The file created is in the form of json with the name rules.json. 
    ```bash
    $ cat > rules.json <<EOF
    [
        {
            "local": [
                {
                    "user": {
                        "name": "{0}"
                    },
                    "group": {
                        "domain": {
                            "name": "Default"
                        },
                        "name": "federated_users"
                    }
                }
            ],
            "remote": [
                {
                    "type": "HTTP_OIDC_ISS"
                }
            ]
        }
    ]
    EOF

    $ openstack mapping create --rules rules.json openidtest_mapping
    ```

4. Create a group for the role assignment on Keystone
    ```bash
    $ openstack group create federated_users
    ```

5. Create a project on Keystone
    ```bash
    $ openstack project create federated_project
    ```

6. Define the roles for group members
    ```bash
    $ openstack role add --group federated_users --project federated_project member
    ```

7. Create a federation protocol for the identity provider based on the mapping rules that has been created. 
    ```bash
    $ openstack federation protocol create openid --mapping openidtest_mapping --identity-provider openidtest
    ```

    <p style="color: orange">* Note: make sure the federation protocol name matches the specified valid keystone authentication rules, such as openid and saml2. </p>

https://docs.openstack.org/keystone/latest/admin/federation/configure_federation.html#sp-prerequisites 

### Configure Apache Authentication Module to OpenID-Connect

1. Create a protected endpoint
    ```conf

    # Keystone
    <Location /v3/OS-FEDERATION/identity_providers/<IDENTITYPROVIDER>/protocols/<PROTOCOL>/auth>
    Require valid-user
    AuthType [...]
    ...
    </Location>

    # Horizon (SSO)
    <Location /v3/auth/OS-FEDERATION/websso/<PROTOCOL>>
    Require valid-user
    AuthType [...]
    ...
    </Location>

    <Location /v3/auth/OS-FEDERATION/identity_providers/<IDENTITYPROVIDER>/protocols/<PROTOCOL>/websso>
    Require valid-user
    AuthType [...]
    ...
    </Location>
    ```

2. Configure the authentication module

    This configuration is for Apache web server on virtualhost keystone, set to OIDC option.
    ```conf
    OIDCClaimPrefix "OIDC-"
    OIDCResponseType "id_token"
    OIDCScope "openid email profile"
    OIDCProviderMetadataURL https://<URL_IDENTITY_PROVIDER>/.well-known/openid-configuration
    OIDCClientID <openid_client_id>
    OIDCClientSecret <openid_client_secret>
    OIDCCryptoPassphrase <random string> # random string
    OIDCRedirectURI https://<URL_HORIZON>/v3/OS-FEDERATION/identity_providers/<IDENTITYPROVIDER>/protocols/<PROTOCOL>/auth
    ```

3. Then add the configuration to `/etc/kolla/config/keystone/wsgi-keystone.conf`.
   ```conf
    <VirtualHost *:5000>
    ….
    ….

    ServerName https://10.20.20.100:5000
    OIDCClaimPrefix "OIDC-"
    OIDCResponseType "id_token"
    OIDCScope "openid email profile"
    OIDCProviderMetadataURL http://10.20.20.14:8081/auth/realms/freeipa-realm/.well-known/openid-configuration 
    OIDCClientID openstack-client # keycloak client id
    OIDCClientSecret xxxx-xxxx-xxxx-xxxx-xxxx # keycloak client secret
    OIDCCryptoPassphrase openstack
    OIDCRedirectURI https://10.20.20.100:5000/v3/OS-FEDERATION/identity_providers/openidtest/protocols/openid/auth 

    # Keystone
    <Location /v3/OS-FEDERATION/identity_providers/openidtest/protocols/openid/auth>
        Require valid-user
        AuthType openid-connect
        LogLevel debug
    </Location>

    # Horizon
    <Location /v3/auth/OS-FEDERATION/websso/openid>
        Require valid-user
        AuthType openid-connect
    </Location>

    <Location /v3/auth/OS-FEDERATION/identity_providers/openidtest/protocols/openid/websso>
        Require valid-user
        AuthType openid-connect
    </Location>

    ...
    ...
    ...
    </VirtualHost>
    ```

    <p style="color:orange">* Note: <br></p>
    -- <b>OIDCProviderMetadataURL</b> is located in `freeipa-realm > Configure > Realm Settings`, then select Endpoints and click "Endpoints OpenID Endpoint Configuration".

### Configure Keystone
This configuration is performed on `/etc/kolla/config/keystone/keystone.conf`. 

1. Add the authentication method.
    ```ini
    [auth]
    methods = password,token,saml2,openid
    ```

2. Configure the remote ID attribute.
    ```ini
    [openid]
    remote_id_attribute = HTTP_OIDC_ISS

    [federation]
    remote_id_attribute = HTTP_OIDC_ISS
    ```

3. Add the Trusted Dashboard (WebSSO). 
    ```ini
    [federation]
    trusted_dashboard = https://10.20.20.100/auth/websso/
    sso_callback_template = /etc/keystone/sso_callback_template.html
    ```

### Configure Horizon

The configuration is performed on `/etc/kolla/config/horizon/local_settings`. 

1. Enable the SSO
    ```python
    WEBSSO_ENABLED = True
    ```

2. Configure the provider for SSO 
    ```python
    WEBSSO_CHOICES = (
        ("credentials", _("Keystone Credentials")),
        ("openidtest_openid", "Keycloak - OpenID Connect"),
    )

    # NOTE: The value is expected to be a tuple formatted as: (<idp_id>, <protocol_id>).
    WEBSSO_IDP_MAPPING = {
        "openidtest_openid": ("openidtest", "openid"),
    }

    ```
### Reconfigure OpenStack
    ```bash
    $ kolla-ansible -i multinode reconfigure
    ```

### Reference
[https://docs.openstack.org/keystone/ussuri/admin/federation/configure_federation.html#create-a-mapping](https://docs.openstack.org/keystone/ussuri/admin/federation/configure_federation.html#create-a-mapping) 

[https://github.com/keycloak/keycloak-documentation/blob/master/securing_apps/topics/oidc/mod-auth-openidc.adoc](https://github.com/keycloak/keycloak-documentation/blob/master/securing_apps/topics/oidc/mod-auth-openidc.adoc) 

[http://wsfdl.com/openstack/2016/02/01/Keystone-Google-Federation-With-OpenID.html](http://wsfdl.com/openstack/2016/02/01/Keystone-Google-Federation-With-OpenID.html) 

[https://www.meshcloud.io/2017/08/25/federated-authentication-with-the-openstack-cli/](https://www.meshcloud.io/2017/08/25/federated-authentication-with-the-openstack-cli/) 

[https://docs.openstack.org/keystone/pike/configuration/samples/keystone-conf.html](https://docs.openstack.org/keystone/pike/configuration/samples/keystone-conf.html)  

[http://www.gazlene.net/federation-devstack.html](http://www.gazlene.net/federation-devstack.html) 
