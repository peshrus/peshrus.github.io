---
layout: post
title:  "Auth0 SP + SSOCircle IdP = SAML Authentication"
date:   2021-04-18 20:12:00 +0100
categories: auth0 sp ssocircle idp saml authentication
---
# Auth0 Service Provider + SSOCircle Identity Provider = Security Assertion Markup Language Authentication

This is a step-by-step-guide how to configure a working Security Assertion Markup Language (SAML)
authentication between Auth0 as a Service Provider (SP) and SSOCircle as an Identity Provider (IdP).
This type of authentication is also known as Single sign-on (SSO).

### Auth0 side

1. Go to **Authentication -> Enterprise**

![Authentication Enterprise](/assets/2021-04-18-auth0-sp-ssocircle-idp-saml-configuration/auth-enterprise.png)

2. Create a new **SAML** connection

![SAML](/assets/2021-04-18-auth0-sp-ssocircle-idp-saml-configuration/saml.png)

3. Set the following fields:
    * **Connection name**: ssocircle (can be any name you want)
    * **Sign In URL**: https://idp.ssocircle.com:443/sso/SSOPOST/metaAlias/publicidp (can be found
      [here](https://www.ssocircle.com/en/idp-tips-tricks/public-idp-configuration/))
    * **X509 Signing Certificate** (can be found
      [here](https://www.ssocircle.com/en/idp-tips-tricks/public-idp-configuration/)):
    ```
    -----BEGIN CERTIFICATE-----
    MIIEYzCCAkugAwIBAgIDIAZmMA0GCSqGSIb3DQEBCwUAMC4xCzAJBgNVBAYTAkRF
    MRIwEAYDVQQKDAlTU09DaXJjbGUxCzAJBgNVBAMMAkNBMB4XDTE2MDgwMzE1MDMy
    M1oXDTI2MDMwNDE1MDMyM1owPTELMAkGA1UEBhMCREUxEjAQBgNVBAoTCVNTT0Np
    cmNsZTEaMBgGA1UEAxMRaWRwLnNzb2NpcmNsZS5jb20wggEiMA0GCSqGSIb3DQEB
    AQUAA4IBDwAwggEKAoIBAQCAwWJyOYhYmWZF2TJvm1VyZccs3ZJ0TsNcoazr2pTW
    cY8WTRbIV9d06zYjngvWibyiylewGXcYONB106ZNUdNgrmFd5194Wsyx6bPvnjZE
    ERny9LOfuwQaqDYeKhI6c+veXApnOfsY26u9Lqb9sga9JnCkUGRaoVrAVM3yfghv
    /Cg/QEg+I6SVES75tKdcLDTt/FwmAYDEBV8l52bcMDNF+JWtAuetI9/dWCBe9VTC
    asAr2Fxw1ZYTAiqGI9sW4kWS2ApedbqsgH3qqMlPA7tg9iKy8Yw/deEn0qQIx8Gl
    VnQFpDgzG9k+jwBoebAYfGvMcO/BDXD2pbWTN+DvbURlAgMBAAGjezB5MAkGA1Ud
    EwQCMAAwLAYJYIZIAYb4QgENBB8WHU9wZW5TU0wgR2VuZXJhdGVkIENlcnRpZmlj
    YXRlMB0GA1UdDgQWBBQhAmCewE7aonAvyJfjImCRZDtccTAfBgNVHSMEGDAWgBTA
    1nEA+0za6ppLItkOX5yEp8cQaTANBgkqhkiG9w0BAQsFAAOCAgEAAhC5/WsF9ztJ
    Hgo+x9KV9bqVS0MmsgpG26yOAqFYwOSPmUuYmJmHgmKGjKrj1fdCINtzcBHFFBC1
    maGJ33lMk2bM2THx22/O93f4RFnFab7t23jRFcF0amQUOsDvltfJw7XCal8JdgPU
    g6TNC4Fy9XYv0OAHc3oDp3vl1Yj8/1qBg6Rc39kehmD5v8SKYmpE7yFKxDF1ol9D
    KDG/LvClSvnuVP0b4BWdBAA9aJSFtdNGgEvpEUqGkJ1osLVqCMvSYsUtHmapaX3h
    iM9RbX38jsSgsl44Rar5Ioc7KXOOZFGfEKyyUqucYpjWCOXJELAVAzp7XTvA2q55
    u31hO0w8Yx4uEQKlmxDuZmxpMz4EWARyjHSAuDKEW1RJvUr6+5uA9qeOKxLiKN1j
    o6eWAcl6Wr9MreXR9kFpS6kHllfdVSrJES4ST0uh1Jp4EYgmiyMmFCbUpKXifpsN
    WCLDenE3hllF0+q3wIdu+4P82RIM71n7qVgnDnK29wnLhHDat9rkC62CIbonpkVY
    mnReX0jze+7twRanJOMCJ+lFg16BDvBcG8u0n/wIDkHHitBI7bU1k6c6DydLQ+69
    h8SCo6sO9YuD+/3xAGKad4ImZ6vTwlB4zDCpu6YgQWocWRXE+VkOb+RBfvP755PU
    aLfL63AFVlpOnEpIio5++UjNJRuPuAA=
    -----END CERTIFICATE-----
    ```
    * **Sign Out URL**: https://idp.ssocircle.com:443/sso/IDPSloPost/metaAlias/publicidp (can be
      found [here](https://www.ssocircle.com/en/idp-tips-tricks/public-idp-configuration/))
   * **Request Template (optional)**:

   ```xml
    <samlp:AuthnRequest xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
        @@AssertServiceURLAndDestination@@
        ID="@@ID@@"
        IssueInstant="@@IssueInstant@@"
        ProtocolBinding="@@ProtocolBinding@@" 
        Version="2.0">
        <saml:Issuer xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion">urn:auth0:YOUR_TENANT:ssocircle</saml:Issuer>
    </samlp:AuthnRequest>
   ```
   **Issuer** is the **Entity ID**, the format can be
   found [here](https://auth0.com/docs/protocols/saml-protocol/saml-identity-provider-configuration-settings#entity-id).

4. Click **"Create"**

5. Go to the **"Mappings"** tab and put the following into the text area:
```json
{
  "email": "EmailAddress",
  "given_name": "FirstName",
  "family_name": "LastName"
}
```

6. Go to the **"Login Experience"** tab

7. **Login Experience Customization -> Home Realm Discovery -> Identity Provider domains**: 
   <`YOU_DOMAIN`>.

   The domain is the one associated with your identity provider. If you specify example.com, all the
   emails ending with this domain will be authorized by your identity provider (SSOCircle in this
   case).

8. Go to the **"Applications"** tab and enable applications you want to have SAML authentication.

9. Go to **Branding -> Universal Login**

![Branding Universal Login](/assets/2021-04-18-auth0-sp-ssocircle-idp-saml-configuration/universal-login.png)

10. Set **Initial Sign-in screen -> Identifier First**

![Identifier First](/assets/2021-04-18-auth0-sp-ssocircle-idp-saml-configuration/identifier-first.png)

### SSOCircle side

1. [Register](https://idp.ssocircle.com/sso/UI/Login)

2. Go to **[Manage Metadata](https://idp.ssocircle.com/sso/hos/ManageSPMetadata.jsp)**

3. Click **[Add new Service Provider](https://idp.ssocircle.com/sso/hos/SPMetaInter.jsp)**:
    * **The FQDN of the ServiceProvider**: auth0.com
    * **Attributes sent in assertion (optional)**: EmailAddress
    * **The SAML Metadata information of your SP**: XML returned
      by https://YOUR_DOMAIN/samlp/metadata?connection=ssocircle 
      (see [here](https://auth0.com/docs/protocols/saml-protocol/saml-identity-provider-configuration-settings#metadata))

---

When the configuration is done the user is redirected to SSOCircle on login to be authenticated.

![SAML SSO](/assets/2021-04-18-auth0-sp-ssocircle-idp-saml-configuration/saml-sso.png)

The user authenticated by SSOCircle is allowed to log in to the configured application. If the user
has not existed before, it's created with the following user_id: **samlp|ssocircle|<`SSOCircle_User`>**, 
where **ssocircle** is **<`YOUR_CONNECTION_NAME`>**.