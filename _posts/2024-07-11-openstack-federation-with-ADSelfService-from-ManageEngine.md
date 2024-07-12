---
layout: post
title: OpenStack federation with ADSelfService Plus
tags: [openstack, cloud, sso, IDP, Active Directory]
category: [openstack]
date: 2024-07-11 21:50 +0200
---


A while ago I encountered a scenario where the request was to use the ADSelfService Plus from Manage Engine to SSO in the OpenStack dashboard. Because it has taken a few iterations I decided to document the process here.

In this case, the OpenStack was deployed with Kolla-ansible and the config will be managed by Kolla. Of course, you can do this manually or with the tool of your choice.

## Starting point (Prerequisites)

In the following configs, there will be 3 test nodes. Those are instances in an OpenStack private cloud and I have floating IPs assigned to them.

| Node | Role                 | IP - internal             | IP - external | Domain             |
| ---- | -------------------- | ------------------------- | ------------- | ------------------ |
| OS   | all-in-one openstack | 192.160.10.18, 10.0.0.240 | 172.30.6.248  | os.gbutnaru.me     |
| AD   | active directory     | 192.160.10.79             | -             | ad.gbutnaru.me     |
| AD4S | IDP                  | 192.160.10.184            | 172.30.6.113  | sso-me.gbutnaru.me |


## Adding the AD domain to the ADSelfService

I've installed the ADSelfService Plus app on a clean Windows Server machine and configured it to be accessible via [sso-me.gbutnaru.me](https://sso-me.gbutnaru.me)

Using the admin account, the AD domain was added from the `Domain Settings`

![ADSelf-add-domain.png](/assets/img/posts/SSO-ME/ADSelf-add-domain.png)

![ADSelf-add-domain-2.png](/assets/img/posts/SSO-ME/ADSelf-add-domain-2.png)

![/assets/img/posts/SSO-ME/ADSelf-add-domain-3.png](/assets/img/posts/SSO-ME/ADSelf-add-domain-3.png)

## Create a custom application for the ADSelfService

From the `Configuration` menu, under `Password Sync/Single Sign-on`, a new application can be added using the `Add application` button.

![/assets/img/posts/SSO-ME/ADSelfService-SSO-1.png](/assets/img/posts/SSO-ME/ADSelfService-SSO-1.png)

In this case, it will be a `Custom Application`

![/assets/img/posts/SSO-ME/ADSelfService-SSO-2.png](/assets/img/posts/SSO-ME/ADSelfService-SSO-2.png)

The following configs will be made:
- `Application Name`
- `Domain Name`
- `Enable OAuth/OpenID Connect`
- `Support SSO Flow` - one can also use IDP Initiated if needed
- `Login Redirect URL(s)` - the URL for the keystone endpoint -  `https://os.gbutnaru.me:5000/redirect_uri`
- `Allow Refresh Token`
- `Key Algorithm` - the default HS256 did not work - `RS512`

Also, we will have to use the information from the `IDP Details` in the OpenStack configs.

![/assets/img/posts/SSO-ME/ADSelfService-SSO-3.png](/assets/img/posts/SSO-ME/ADSelfService-SSO-3.png)

Next, let's see what user attributes are the IDP able to send by clicking the `Advanced` button from the above image or the applications listing:
![/assets/img/posts/SSO-ME/ADSelfService-SSO-4.png](/assets/img/posts/SSO-ME/ADSelfService-SSO-4.png)

Let's change the source attribute for the username and leave the rest as it is for now:
![/assets/img/posts/SSO-ME/ADSelfService-SSO-5.png](/assets/img/posts/SSO-ME/ADSelfService-SSO-5.png)
![/assets/img/posts/SSO-ME/ADSelfService-SSO-6.png](/assets/img/posts/SSO-ME/ADSelfService-SSO-6.png)
## Configure the OpenStack keystone

As I said, OpenStack is deployed on an all-in-one node, with Kolla Ansible and the configs will be deployed using the same tool.

For the sake of simplicity and more clarity, I like to keep the configs in different files and, don't want to use the `globals.yml` for everything.

Inside `/etc/kolla/config/` create `globals.d` directory with `keystone.yml` file with the following content:
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-1.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-1.png)

With the above config, we are telling kolla to create an identity provider and a mapping. [You can do it manually by following the docs](https://docs.openstack.org/keystone/2024.1/admin/federation/configure_federation.html#create-an-identity-provider)

Most of the configs are self-explanatory but it's important to mention that you have to use your own `identifier (Issuer)` and `keystone_federation_oidc_jwks_uri (Keys Endpoint URL)` that you can find under the `IDP Details` on the ADSelfService portal:
![/assets/img/posts/SSO-ME/ADSelfService Plus-SSO-4.png](/assets/img/posts/SSO-ME/ADSelfService%20Plus-SSO-4.png)
![/assets/img/posts/SSO-ME/ADSelfService Plus-SSO-5.png](/assets/img/posts/SSO-ME/ADSelfService%20Plus-SSO-5.png)

Next, create the folder for the IDP configs (`metadata_folder` and `mappings`) and the files with the provider metadata, client credentials, and additional configs.

![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-2.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-2.png)

The names of the files will be formed using the `issuer` with `.conf`, `.provider`, and `.client` suffixes. **The name of the files needs to be URL-safe ( you can replace `/` with `%2F`).**

All the files are in the JSON format:
- the `.conf` file in this case is just an empty JSON
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-3.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-3.png)
- the `.client` file contains the `client ID` and the `client Secret` and serves as a means of confirming the client's authenticity,  for example:
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-4.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-4.png)
- the `.provider` contains the metadata of the identity provider, which you can find under the `IDP Details` -> `Well-Known Configuration` on the ADSelfService portal, for example:
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-5.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-5.png)

Besides the metadata folder created previously, in the `keystone.yml` there is a reference to the `/etc/kolla/config/idp_integration/mappingSSO`.  For this scenario, the following mapping creates a project for every user authenticated through SSO and assigns the `member` role. 

![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-6.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-6.png)
For more information about the mappings check the [docs](https://docs.openstack.org/keystone/2024.1/admin/federation/mapping_combinations.html)

The final step in this scenario is to run the playbook called `reconfigure`, targeting the `horizon` and `keystone` components:
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-7.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-7.png)

## Test the SSO integration

After the reconfiguration of the OpenStack environment, when navigating to the Horizon dashboard, you should be able to see the dropdown with the new authentication method.

![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-8.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-8.png)

Choosing the new option you are going to be redirected to the ADSelfService portal to log in and then back to the horizon:
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-10.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-10.png)
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-11.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-11.png)

The above pop-up is caused by the fact that the horizon is accessed through HTTP, not HTTPS. I am going to write a separate blog post about how to secure Horizon with TLS.

Sadly, after pressing `Continue`, there is an error:
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-9.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-9.png)

After some debugging, I found and fixed the first problem. It looks like the OpenStack expected the response mode to be `fragment` but the IDP uses `form_post` as you can see in `/var/log/kolla/keystone/keystone-apache-public-error.log` :

```
oidc_proto_validate_response_mode: requested response mode (fragment) does not match the response mode used by the OP (form_post)
```


It is possible to force the response mode in the configuration. You can do it manually by adding `OIDCResponseMode "form_post"` in `/etc/kolla/keystone/wsgi-keystone.conf` and restarting the keystone container (`docker restart keystone`).

The second problem was the self-signed certificate used for the IDP (ADSelf Service Plus). 
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-12.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-12.png)

Being a demo environment, I added `OIDCSSLValidateServer "Off"` in `/etc/kolla/keystone/wsgi-keystone.conf` followed by the restart of the keystone container.

Lastly, similar to before, I specified the signature algorithm with `OIDCIDTokenSignedResponseAlg RS512` to avoid the following error: `Invalid Key Algorithm found. Please choose a valid RS signing algorithm.`
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-13.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-13.png)

Although some progress was made, another error arose:

![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-14.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-14.png)

Searching again in logs, it looks like something is wrong with the mapping: 
```
Could not map any federated user properties to identity values. Check debug logs or the mapping used for additional details.
```

![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-15.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-15.png)

The IDP doesn't send the username in the expected attribute (uses `name` instead of `username`). For this reason, you need to change the mappings as follows:
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-16.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-16.png)

You can see and modify the mappings created by the kolla ansible using the OpenStack client by running `openstack mapping list` and  `openstack mapping set --rules /etc/kolla/config/idp_integration/mappingSSO mappingSSO`:
![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-17.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-17.png)

After the mapping modification, you can successfully log in to the horizon via the ADSelfService Plus IDP using the account created in the Microsoft Active Directory.

![/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-18.png](/assets/img/posts/SSO-ME/openstack-kolla-keystone-federation-18.png)

All the manual changes to the `/etc/kolla/keystone/wsgi-keystone.conf` file should be added to the kolla ansible config for the keystone component. The final config should be something similar to the following:

```
keystone_identity_providers:

  - name: "ManageEngine"
    openstack_domain: "SSO-ME"
    protocol: "openid"
    identifier: "https://sso-me.gbutnaru.me/sso/oauth/269fb51d4d7abe707447a8786d6da91735130dd8"
    public_name: "Log in via ADSelfService Plus"
    attribute_mapping: "mappingSSO"
    metadata_folder: "/etc/kolla/config/idp_integration/idp_metadata"

keystone_identity_mappings:
  - name: "mappingSSO"
    file: "/etc/kolla/config/idp_integration/mappingSSO"

keystone_federation_oidc_jwks_uri: "https://sso-me.gbutnaru.me/sso/oauth/269fb51d4d7abe707447a8786d6da91735130dd8/keys"

keystone_federation_oidc_additional_options:
    OIDCResponseMode: form_post
    OIDCSSLValidateServer: Off
    OIDCIDTokenSignedResponseAlg: RS512
```

## Takeaways

After some trial and error, the Manage Engine product - ADSelf Service Plus - could be used as an Identity provider to Single Sign On into the OpenStack dashboard.

For debugging purposes and tests, it helped to enable debug mode on the keystone by adding `debug = True` on the keystone config `/etc/kolla/keystone/keystone.conf`. 

To avoid waiting for the kolla to reconfigure the environment on every change, they were manually added to the configs, and the container restarted.
