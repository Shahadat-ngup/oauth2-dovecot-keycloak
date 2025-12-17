# Configuring OAuth2 Authentication for Mail Server with Keycloak, Dovecot, Postfix, and Roundcube

## System Overview

Successfully configured OAuth2 authentication for a complete mail server stack using:

- **Dovecot**: 2.4.2-1+debian12 (0962ed2104)
- **Postfix**: 3.7.11-0+deb12u1
- **Keycloak**: 26.4.7
- **Roundcube**: 1.7-beta2
- **OpenLdap**: slapd 2.5.13+dfsg-5

## Authentication Flow

```
User (Browser) → Roundcube → Keycloak (OAuth2 + MFA)
                     ↓
              OAuth2 Token
                     ↓
         ┌───────────┴───────────┐
         ↓                       ↓
    IMAP (Dovecot)          SMTP (Postfix)
         ↓                       ↓
    XOAUTH2 Auth            XOAUTH2 Auth
         ↓                       ↓
    Token Introspection ←───────┘
         ↓
    Keycloak Validation
         ↓
    LDAP UserDB (home, uid, gid, quota)
```

## Working Configuration

### Authentication Mechanisms

The mail server supports dual authentication:
- **PLAIN/LOGIN**: Traditional username/password (via LDAP)
- **XOAUTH2**: OAuth2 bearer tokens (via Keycloak)

**Important**: Use `XOAUTH2` instead of `OAUTHBEARER` for SMTP compatibility.

### Dovecot Configuration

#### /etc/dovecot/conf.d/10-auth.conf
```conf
auth_mechanisms = plain login xoauth2
!include auth-oauth2.conf.ext
```

#### /etc/dovecot/conf.d/auth-oauth2.conf.ext
```conf
# Global OAuth2 Settings
oauth2_introspection_url = https://dovecot:CLIENT_SECRET@your_keycloak_server/realms/ccom/protocol/openid-connect/token/introspect
oauth2_introspection_mode = post
oauth2_username_attribute = email
oauth2_active_attribute = active
oauth2_active_value = true
oauth2_force_introspection = yes
oauth2_issuers = https://your_keycloak_server/realms/ccom
oauth2_token_expire_grace = 1min

# PassDB OAuth2
passdb oauth2 {
    # CRITICAL: Use XOAUTH2 only (OAUTHBEARER has authzid issues with SMTP)
    mechanisms_filter = xoauth2
}
```

**Key Points:**
1. **Embedded Credentials**: Client credentials are embedded in the introspection URL (`https://user:pass@host`) to work around Dovecot 2.4 not sending client_id/client_secret properly
2. **Email as Username**: `oauth2_username_attribute = email` ensures username matches user login
3. **XOAUTH2 Only**: Using only XOAUTH2 mechanism to avoid SMTP authentication issues

### Postfix Configuration

#### /etc/postfix/main.cf
```conf
smtpd_sasl_type = dovecot
smtpd_sasl_path = /var/spool/postfix/private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname
```

#### /etc/postfix/master.cf
```conf
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
```

### Roundcube Configuration

#### /etc/roundcube/config.inc.php
```php
<?php
// OAuth2 Configuration
$config['oauth_provider'] = 'generic';
$config['oauth_provider_name'] = 'IPB Keycloak';
$config['oauth_client_id'] = "your_client_id";
$config['oauth_client_secret'] = "YOUR_CLIENT_SECRET";
$config['oauth_auth_uri'] = 'https://your_keycloak_server/realms/ccom/protocol/openid-connect/auth';
$config['oauth_token_uri'] = 'https://your_keycloak_server/realms/ccom/protocol/openid-connect/token';
$config['oauth_identity_uri'] = 'https://your_keycloak_server/realms/ccom/protocol/openid-connect/userinfo';
$config['oauth_scope'] = "openid email profile";

// CRITICAL: Tell Roundcube to use 'email' field as username
$config['oauth_identity_fields'] = ['email'];

// REQUIRED: Fixed IMAP and SMTP hosts for OAuth2
$config['smtp_server'] = 'tls://mail.ccom.ipb.pt';

// RECOMMENDED: Cache tokens in database
$config['oauth_cache'] = 'db';
```

### Keycloak Configuration

1. **Dovecot Client** (for token introspection):
   - Client ID: `dovecot`
   - Service Account Enabled: `ON`
   - Standard Flow: `ON`
   - Standard Token Exchange: `ON`
   - OAuth 2.0 Device Authorization Grant: `ON`


2. **Webmail Client** (for Roundcube):
   - Client ID: `roundcube_id`
   - Valid Redirect URIs: `https://your_roundcube_server/index.php/login/oauth`
   - Web Origins: `https://your_roundcube_server`
   - Standard Flow: `ON`
   - Standard Token Exchange: `ON`
   - OAuth 2.0 Device Authorization Grant: `ON`

## OAUTHBEARER vs XOAUTH2 Issue

### Problem with OAUTHBEARER

When using `OAUTHBEARER`, SMTP authentication failed with:

```
Dec 16 15:05:32 auth(?,193.136.195.164,sasl:oauthbearer): Info: sasl(oauthbearer): Invalid gs2-header in request: Invalid authzid field
```

**Root Cause:** Roundcube sends OAUTHBEARER requests with an authorization identity (authzid):
```
n,a=hosain@ipb.pt\x01auth=Bearer eyJhbGci...
```

The `a=hosain@ipb.pt` field is the authzid. While Dovecot IMAP accepts this, **Dovecot SMTP via Postfix SASL rejects it**.

### Working Solution: XOAUTH2

Switching to `XOAUTH2` mechanism resolved the issue:
- XOAUTH2 uses a simpler format without authzid field
- Both IMAP and SMTP authentication work correctly

**Note:** The OAUTHBEARER issue might be fixable with additional Dovecot configuration, but this requires further debugging. XOAUTH2 is the recommended approach for now.

## Successful Authentication Logs

### IMAP Login (Working)
```log
Dec 16 15:19:23 auth(hosain,193.136.195.194,sasl:xoauth2): Debug: oauth2: Introspection succeeded
Dec 16 15:19:23 auth(hossain,193.136.195.194,sasl:xoauth2): Debug: oauth2 active_attribute check succeeded
Dec 16 15:19:23 auth(hossain,193.136.195.194,sasl:xoauth2): Debug: ldap: Finished userdb lookup
Dec 16 15:19:23 auth: Debug: master userdb out: USER hosain uid=2000 gid=996 home=/home/vmail/ipb/hosain/Maildir/ quota_storage_size=104857600B auth_mech=XOAUTH2
Dec 16 15:19:23 imap-login: Info: Logged in: user=<hosain>, method=XOAUTH2, rip=193.136.195.164, lip=193.136.194.121, TLS, session=<E8uwQBNG4pTBiMOk>
```

### SMTP Authentication (Working)
```log
Dec 16 15:19:37 auth: Debug: conn unix:auth (pid=106274,uid=102): client in: AUTH 1 XOAUTH2 service=smtp
Dec 16 15:19:37 auth(?,193.136.195.164,sasl:xoauth2): Debug: oauth2: Introspection succeeded
Dec 16 15:19:37 auth(?,193.136.195.164,sasl:xoauth2): Debug: oauth2 active_attribute check succeeded
Dec 16 15:19:37 auth: Debug: conn unix:auth: client passdb out: OK 1 user=hosain@ipb.pt
```

**Note:** No FAIL message indicates successful authentication. The email is then sent successfully through Postfix submission service.

## Conclusion

Successfully implemented OAuth2 authentication for a complete mail server stack with:
- ✅ Single Sign-On via Keycloak
- ✅ Multi-Factor Authentication support
- ✅ Both IMAP and SMTP OAuth2 authentication
- ✅ Backward compatibility with traditional password authentication
- ✅ User information from LDAP (home, uid, gid, quota)

The key was using **XOAUTH2** instead of OAUTHBEARER for SMTP compatibility.

---

**Author**: Shahadat Hossain  
**Date**: December 16, 2025  
**Environment**: Debian 12 (Bookworm)
