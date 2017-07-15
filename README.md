Federated Atlassian Confluence
==============================

An Ansible role for fully automated deployment of Atlassian Confluence with federated authentication, based on a zipped backup file.
This role was created because Atlassian does not offer a good way to deploy Confluence; their upgrade and installation paths contain many tedious browser clicking steps: https://jira.atlassian.com/browse/CONF-45381.
This roles automates that using a sequence of HTTP requests to localhost.


Requirements
------------

This role assumes you want to run a production instance, so **you need a valid license** for this to work. See the example playbook on how to configure this. You can get free one-month trial licenses, see https://confluence.atlassian.com/display/DOC/Start+your+trial.

The role installs OpenJDK which is [not officially supported by Atlassian](https://confluence.atlassian.com/confkb/is-openjdk-supported-by-confluence-297667642.html). It runs fine, but if you need "support" you should use Oracle JDK installed.

The role uses Ubuntu 16.04 and PostgreSQL, but with some minor adjustments you should be able to use other distros and databases. Since it will restore a backup file, there is no _need_ to stick to a different existing database though.

Role Variables
--------------

`atlassian_confluence_version` is the verion you want to install. 

`atlassian_confluence_baseurl` is the URL where to retrieve the *tar.gz* files from. Defaults to the official Atlassian web site. Since the tarballs are quite large, it can make sense to mirror them locally. Note that Atlassian [does not provide any checksums for their products](https://jira.atlassian.com/browse/JRASERVER-63158), so keep your fingers crossed.

`atlassian_confluence_basedir` is path where to download and extract the *tar.gz* file. Defaults to `/opt/atlassian`.

`atlassian_confluence_home` is the `confluence.home` variable. Defaults to `/home/confluence`.

`atlassian_confluence_user`, `atlassian_confluence_uid`, `atlassian_confluence_group`, `atlassian_confluence_gid` are variables to set up a dedicated user/group to run the application. Default to `confluence`, `confluence`, 10000, and 10000.

`atlassian_confluence_server_xml` is list of changes to `server.xml` It uses XPath to edit/add/remove exiting properties. Defaults to settings for an HTTPS set-up:

    atlassian_confluence_server_xml:
    - xpath: /Server/Service/Connector
      ensure: present
      attribute: proxyPort
      value: 443
    - xpath: /Server/Service/Connector
      ensure: present
      attribute: scheme
      value: https

`atlassian_confluence_catalina_opts` is the list of custom *CATALINA_OPTS* properties. At the moment you can't change the existing set - only append it.


Dependencies
------------

* The `cmprescott.xml` role to manipulate XML files.
* The `dnmvisser.mellon_sp` role to deploy mod_mellon.
* The `geerlingsguy.apache` role to deploy Apache 2.4.
* A SAML2 IdP.
* An X-509 key/cert for use with Apache.



Example Playbook
----------------


```yaml
- hosts: all 
  name: Provision federated Confluence instance
  become: true
  roles:
    - cmprescott.xml
    - confluence
    - mellon_sp
    - geerlingguy.apache

  vars:
    # First one is the main vhost, will be used by apache and confluence
    vhosts:
      - wiki.example.org
      - some.other.host.name

    atlassian_confluence_version: 6.3.1
    atlassian_confluence_admin_username: visser
    atlassian_confluence_admin_password: hackme
    atlassian_confluence_server_id: ABCD-EFGH-1234-5678
    atlassian_confluence_license: |
      AAABLg0ODAoPeNptUNtKw0AQfd+vWPBFH7bkYptYWLAmQULTtMh6QXzZxmldSDdldhPtJ/hd/pibR
      rGKDwPDnMucmZN7eKZ53VLPp87F2HfEQ4dGxZJ6jjsiMegS5c7IWvGoVuuqAVUCPS0AW8CzpzFNW
      lE1oiOQCOHQxMIA7+TMdZl3TkorHIjSyBa4wQb6QWEEGkC+FpUGYs2NpSQzISv+Cqut0Ba8tAVKD
      GrckJ9FRyaZLEFpWO53kIst8Gg+myU2UTrJSNVDd4C603jEWitj3ewBydtO4v4op8c8n8xxI5TU/
      Y7rj/dJviT9oWnMr4KHRzYKiyGbpoXPFtOJQ2ok57ZYFoZB4AQh+Ypj6Vka/0IOafNmuwKcr2+1D
      cWZ+83/P86iwfJFaPj7zk9Y3Iz1MCwCFGap3PR0egsVCBJgBzVWbbqPMJaZAhQmoxJsgl8AvZpdd
      4GWqSz40BlDEQ==X02f9

    # This has to be on the same file system as the Confluence home directory
    atlassian_confluence_backup_zip: /opt/confluence-backup.zip

    atlassian_confluence_custom_html_body_begin: '&lt;div style="background-color: #ff9966; text-align: center; font-size: 12px;"&gt;This wiki has been upgraded recently to version {{ atlassian_confluence_version }}.&lt;/div&gt;'

    # These values will be used to construct the 'Server Base URL'
    atlassian_confluence_server_xml:
      - xpath: /Server/Service/Connector
        ensure: present
        attribute: proxyPort
        value: 443
      - xpath: /Server/Service/Connector
        ensure: present
        attribute: scheme
        value: https
      - xpath: /Server/Service/Connector
        ensure: present
        attribute: proxyName
        value: "{{ vhosts[0] }}"
    atlassian_confluence_catalina_opts:
      - '-Dhttp.proxyHost=proxy.geant.org'
      - '-Dhttp.proxyPort=80'

    plugins:
      - com.balsamiq.confluence.plugins.mockups
      - com.gliffy.integration.confluence
        #- com.atlassian.confluence.extra.team-calendars


    # Mod_mellon SP
    mellon_sp_remote_idp_metadata_url: "https://login.terena.org/idp/saml2/idp/metadata.php"
    mellon_sp_saml_uid_attribute: 'uid'

    mellon_sp_crt: |
      -----BEGIN CERTIFICATE-----
      MIID+DCCAuCgAwIBAgIJAMzr8kq0xas6MA0GCSqGSIb3DQEBCwUAMIGQMQswCQYD
      VQQGEwJOTDELMAkGA1UECAwCTkgxEjAQBgNVBAcMCUFtc3RlcmRhbTETMBEGA1UE
      CgwKRXhhbXBsZU9yZzEOMAwGA1UECwwFU0FNTDIxGTAXBgNVBAMMEHdpa2kuZXhh
      bXBsZS5vcmcxIDAeBgkqhkiG9w0BCQEWEWFkbWluQGV4YW1wbGUuY29tMB4XDTE3
      MDcxNDIyNTQ1MFoXDTM4MTEyMTIyNTQ1MFowgZAxCzAJBgNVBAYTAk5MMQswCQYD
      VQQIDAJOSDESMBAGA1UEBwwJQW1zdGVyZGFtMRMwEQYDVQQKDApFeGFtcGxlT3Jn
      MQ4wDAYDVQQLDAVTQU1MMjEZMBcGA1UEAwwQd2lraS5leGFtcGxlLm9yZzEgMB4G
      CSqGSIb3DQEJARYRYWRtaW5AZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3DQEBAQUA
      A4IBDwAwggEKAoIBAQDe8JWwmoCSiErIktjBoAgBPnbKrDNXb8vRWNFO/E9+z0rf
      GKpd7m814r2kAkosmZ7PHGt27n6uy3wpSSoLnzIlUqbzDIMMMgztN/Kx9bSGmoT4
      OfnlHxBcHOHAdKNjSVIktBfYl6D2fRGBEhiqApUQfcHl8Us+XXqU+83ggL26DQG9
      VnIwmkA3SygTciG4DtIsgeAIVU2Q5HoMqecqCRSoANUzf1Haa7nMk3dY6LID/Xur
      q3urrw61ivQlaPlVOgeXt75W1N5HkraqAZfomJn8W4vLv8Vs8c8W+mRuTJnN8JIf
      Zf+0+MOpWpCYVr/p3ZQgCPVb1UHY0jVhbb4c3BrzAgMBAAGjUzBRMB0GA1UdDgQW
      BBTwzZMvj8OQMkvwc5yucjtT4AOfVDAfBgNVHSMEGDAWgBTwzZMvj8OQMkvwc5yu
      cjtT4AOfVDAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBCwUAA4IBAQBY55KU
      6I1KAl70WlDGZHUZHl6EXjkarF7YVZPkCg2hfBpONQkI0OYes4rEEODKMoUfMHM+
      ZsXEq/fAzqIU9GNOfUGGBEvsMC3dxiGXdp8KV0URT6xTrezw0c5APBCHxlfibQ9H
      ypab++KxS4hxv3ghlBu0VEAba/Q1i33EFXiJmwtgl8jKLpAkJuj5C3WfWFrSpzz/
      bG1HkYfeLpOtPuBWTGgqxL4LmYZu7mRiRzxxBrV4XU8agxNQqJOJBtugF38LmW50
      bgUol0ca47SJNAJxMiwFx2sTXz8Z9OtSEWp3O7rAv3tPNcevXJ/qSEWUD2sO4VUW
      gqKX31cWQLRqX1PJ
      -----END CERTIFICATE-----
    
    mellon_sp_key: |
      -----BEGIN PRIVATE KEY-----
      MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDe8JWwmoCSiErI
      ktjBoAgBPnbKrDNXb8vRWNFO/E9+z0rfGKpd7m814r2kAkosmZ7PHGt27n6uy3wp
      SSoLnzIlUqbzDIMMMgztN/Kx9bSGmoT4OfnlHxBcHOHAdKNjSVIktBfYl6D2fRGB
      EhiqApUQfcHl8Us+XXqU+83ggL26DQG9VnIwmkA3SygTciG4DtIsgeAIVU2Q5HoM
      qecqCRSoANUzf1Haa7nMk3dY6LID/Xurq3urrw61ivQlaPlVOgeXt75W1N5Hkraq
      AZfomJn8W4vLv8Vs8c8W+mRuTJnN8JIfZf+0+MOpWpCYVr/p3ZQgCPVb1UHY0jVh
      bb4c3BrzAgMBAAECggEAFWn4huEgTnLm9AMe7OJJQo1Ubb1CpTho1G/6kuKrZBvX
      Ljy5SJJ5fiyMWK+bnlMXLP+m1uKrqnCDOZf8gOdGss0QmjHueIJqOXLxTeSy9Wbs
      NMek2Dw2nxJrIMhTVVsal8nwrG5MGMEEqGgrlFDxLodV05dsyc3C04KoUNMs5izh
      tQDP9WZ3uRetsjOhbGM0X5SsgeSKD/K8t67IR//tqVslCIDrFylN6VX9zvw/szOJ
      jwUaXXbNtZ5q+x3UhxitgrmTZdlx2UlKsIwiVKvzAFj5sJyHap4Wx60tHAPaQUsM
      RSmzREAmkS7+otUV+NqIWA2KDlMsj3TGpjPl5LkouQKBgQD7P8wEIDvZSt3afOON
      sGIWMyaSmplEC5e0rT/O/z49b3RkQ/JlBw38AFN/+CIlPA3F3pb0/e1MRY7ktvVn
      FfD1Ih2067gjeOkX2+2GUXoi7csd6wdozugN3dPWe+ZPAZoIee3iALjQfsAJhRPQ
      MjR6zOB7GuO6dQxwr8v75R3U3QKBgQDjJ8CkD7NkN3t2v4lEHkKaRQlQVWnkKxuz
      A8VaYdjieWFHBazn8T2SGGH/tztReEzjcpwDdpr0ub/uBZ59WOW89ec319JxafcC
      z+pjyV9NR5SdzPJu88jf2u0uiKqaIkvRvaBwv56haWleSwcKpD3kZJvtY1asP88e
      FMwCCjwKDwKBgQD4Js3BMXkLJ9exSXKS6igm1wR8/fbs7yo6SHdiYlU95owlz7pk
      MDCOul++X/yRnBvaV/vvI7GxiG4W1eHRnCkuZDDFFZ/8YRqP9ydFZ8njH8xi01Sx
      HtKxh2wIRr11QRa60nqyopXnT5kqjebA1iVhOGNrE2bjLWJJFki5lVT+fQKBgCRL
      Cq1r0Ma3UqOjcKQQjaWmWBmcvAF3ncJZXOPW9Fci+5bkYj7gvWylNqsrtG+M4AQL
      IMAw6AsnznGSNkNiu7oYM+wpMHdsbcnmPFXbK73mLOmVgDqX+sVBblnb/h4IUsLd
      IFBDXW3+sJYfOK3LlhPyuzMPhx6YR1hQDFRbHEKjAoGAZehsspBOVxUodN8HNO5A
      sQUAxH68emMi9mc3d9lRL46UI/ApVq+RlrfjUVX/8dxIyPuXiWVw41vs/jITovdt
      Q+I1h/GiOZzmtD+jY4LC3W4X2OlNxLS+AThjDDycGkRBDobz9xekdiKKfSwFLtv/
      qmc9OzmMoTLiuApoLNzWOLY=
      -----END PRIVATE KEY-----


    # Apache & vhost settings
    #
    apache_global_vhost_settings: |
      ServerName {{ vhosts[0] }}
      Serveradmin webmaster@example.com
      ServerTokens ProductOnly
      Header always unset "X-Powered-By"
      CustomLog ${APACHE_LOG_DIR}/other_vhosts_access.log vhost_federated_combined
      # Sortable log format, with federated username.
      LogFormat {% raw %}"%v:%p %{%F %T}t.%{msec_frac}t %h %u \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\""{% endraw %} vhost_federated_combined
    apache_create_vhosts: true
    apache_remove_default_vhost: true
    apache_mods_enabled:
      - auth_mellon.load
      - rewrite.load
      - ssl.load
      - headers.load
      - proxy_http.load
      - proxy.load
      - proxy_wstunnel.load
    apache_conf_disabled:
      - other-vhosts-access-log.conf
    apache_vhosts:
      - servername: "{{ vhosts[0] }}"
        serveralias: "{{ vhosts[1:] | join(' ')}}"
        documentroot: "/var/www/html"
        # See https://github.com/geerlingguy/ansible-role-apache/issues/98
        extra_parameters: |
          # Force everything to HTTPS and configure the rest there
            RewriteEngine On
            RewriteCond %{HTTPS} !=on
            RewriteRule . https://{{ vhosts[0] }}%{REQUEST_URI}

    apache_vhosts_ssl:
      - servername: "{{ vhosts[0] }}"
        serveralias: "{{ vhosts[1:] | join(' ')}}"
        certificate_key_file: /etc/ssl/private/server.key
        certificate_file: /etc/ssl/certs/server.crt
        certificate_chain_file: /etc/ssl/certs/chain.crt
        extra_parameters: |
          ErrorLog ${APACHE_LOG_DIR}/ssl.log
            LogLevel debug
            # Force everything to main vhost
            RewriteEngine On
            RewriteCond %{HTTP_HOST} !^{{ vhosts[0] }}$
            RewriteRule ^/(.*) https://{{ vhosts[0] }}/$1

            # MAJOR SUCKAGE WARNING ON MOBILE
            # The mobile theme does NOT allow editing. Yes you heard it right.
            # There are native apps for iOS/Android but those only support the
            # cloud based version of Confluence, and not the server version (this one).
            RewriteCond %{REQUEST_URI}           ^/m/login/mellon/login
            RewriteRule ^/m/login/mellon/login   /mellon/login [R]

            # Needed to unescape URL below
            AllowEncodedSlashes NoDecode

            RewriteCond %{REQUEST_URI}         ^/mellon/
            RewriteRule ^/mellon/%2Fm%2F(.*)   /m/$1 [R,L]

            ProxyRequests Off
            <Proxy http://[::1]:8090>
              Require all granted
            </Proxy>

            # We need to process this ourselves
            ProxyPass /mellon/ !

            # This is needed for the new Collaborative Editing feature
            # See https://confluence.atlassian.com/confkb/unable-to-edit-any-pages-in-confluence-6-with-error-message-loading-the-editor-s-taking-longer-than-usual-867362716.html
            ProxyPass /synchrony http://[::]:8091/synchrony
            <Location /synchrony>
              Require all granted
              RewriteEngine on
              RewriteCond %{HTTP:UPGRADE} ^WebSocket$ [NC]
              RewriteCond %{HTTP:CONNECTION} Upgrade$ [NC]
              RewriteRule .* ws://{{ vhosts[0] }}:8091%{REQUEST_URI} [P]
            </Location>

            # Proxy to confluence
            ProxyPass / http://[::1]:8090/
            ProxyPassReverse / http://[::1]:8090/

            # Security related
            #Header always set Strict-Transport-Security "max-age=31536000"

            # Force secure cookies
            Header always edit Set-Cookie ^(.*)HttpOnly$ "$1HttpOnly; Secure"

            <Location />
              MellonEnable "info"
              MellonSecureCookie On
              MellonSessionDump Off
              MellonSamlResponseDump Off
              MellonEndpointPath "/mellon"
              MellonUser "{{ mellon_sp_saml_uid_attribute }}"
              MellonSPPrivateKeyFile {{ mellon_sp_cfg_dir }}/sp.key
              MellonSPCertFile {{ mellon_sp_cfg_dir }}/sp.crt
              MellonIdPMetadataFile {{ mellon_sp_cfg_dir }}/idp.xml
              # First unset to avoid security holes
              RequestHeader unset REMOTE_USER
              RequestHeader set REMOTE_USER "%{MELLON_{{ mellon_sp_saml_uid_attribute }}}e" env=MELLON_{{ mellon_sp_saml_uid_attribute }}

              RequestHeader unset CONF_FULLNAME
              RequestHeader set CONF_FULLNAME "%{MELLON_displayName}e" env=MELLON_displayName

              RequestHeader unset CONF_EMAIL
              RequestHeader set CONF_EMAIL "%{MELLON_mail}e" env=MELLON_mail
            </Location>
```

License
-------

MIT

Author Information
------------------

Dick Visser <dick.visser@geant.org>
