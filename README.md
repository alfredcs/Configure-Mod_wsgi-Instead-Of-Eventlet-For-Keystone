Starting with Kilo release, eventlet is replaced with mod_wsgi in front of Keystone service to avoid close_wait staus in TCP sessions.  Per Morgan Fainbery, the chief developer of Kerystone. The main reasons are that

    "Keystone relies on apache/web-server modules to handle federated identity (validation of SAML, etc) and similar SSO type authentication (Kerberos)."
    "Eventlet has proven problematic when it comes to workloads within Keystone, notably that a number of actions cannot yield (either due to lacking in Eventlet, or that the dependent library uses C-bindings that eventlet is not able to work with)."


Followings configuration steps are for RedHat/CentOS platform. Please adjust accordingly for debian or other Linux flavors.
 
Step 1: Create a /etc/httpd/conf.d/wsgi-keystone.conf

Use port 5001 for public and internal auth port and 35358 for admin auth port.

    Listen 5001
    Listen 35358
    <VirtualHost *:5001>
        DocumentRoot "/var/www/keystone"
        <Directory "/var/www/keystone">
            Options Indexes FollowSymLinks MultiViews
            AllowOverride None
            Require all granted
        </Directory>
        WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone display-name=%{GROUP}
        WSGIProcessGroup keystone-public
        WSGIScriptAlias / /var/www/keystone/main
        WSGIApplicationGroup %{GLOBAL}
        WSGIPassAuthorization On
        <IfVersion >= 2.4>
          ErrorLogFormat "%{cu}t %M"
        </IfVersion>
        ErrorLog /var/log/httpd/keystone.log
        CustomLog /var/log/httpd/keystone_access.log combined
    </VirtualHost>
    <VirtualHost *:35358>
        DocumentRoot "/var/www/keystone"
        <Directory "/var/www/keystone">
            Options Indexes FollowSymLinks MultiViews
            AllowOverride None
            Require all granted
        </Directory>
        WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone display-name=%{GROUP}
        WSGIProcessGroup keystone-admin
        WSGIScriptAlias / /var/www/keystone/admin
        WSGIApplicationGroup %{GLOBAL}
        WSGIPassAuthorization On
        <IfVersion >= 2.4>
          ErrorLogFormat "%{cu}t %M"
        </IfVersion>
        ErrorLog /var/log/httpd/keystone.log
        CustomLog /var/log/httpd/keystone_access.log combined
    </VirtualHost>

 
Step 2: Modify /etc/keystone/keystone.conf

In this case, port 5000 is mapped to 5001 on one or multiple nodes running keystone-all via haproxy. Same for 35357. The conf example uses memcache for token persistance.

    [DEFAULT]
    verbose=True
    debug=True
    log_file = /var/log/keystone/keystone.log
    log_dir = /var/log/keystone
    log_format = %(asctime)s %(levelname)8s [%(name)s] %(message)s
    log_date_format = %Y-%m-%d %H:%M:%S
    default_log_levels = amqp=WARN, amqplib=WARN, boto=WARN, qpid=WARN, sqlalchemy=WARN, suds=INFO, oslo.messaging=INFO, iso8601=WARN, requests.packages.urllib3.connectionpool=WARN, urllib3.connectionpool=INFO, websocket=WARN, keystonemiddleware=INFO, routes.middleware=WARN, stevedore=WARN
    admin_token = <token_string>
    public_port = 5000
    admin_port = 35357
    tcp_keepalive = True
    tcp_keepidle = 10
    public_workers=2
    admin_workers=2
    [database]
    connection = mysql://keystone:fc2ac07240830c19d6f4@10.0.0.241/keystone
    [assignment]
    [cache]
    [catalog]
    [credential]
    [ec2]
    [endpoint_filter]
    [endpoint_policy]
    [federation]
    [identity]
    [identity_mapping]
    [kvs]
    [ldap]
    [matchmaker_redis]
    [matchmaker_ring]
    [memcache]
    servers = 10.0.0.241:11211
    [oauth1]
    [os_inherit]
    [paste_deploy]
    config_file=keystone-paste.ini
    [policy]
    [revoke]
    [saml]
    [signing]
    [eventlet_server_ssl]
    enable = False
    cert_required = False
    [auth]
    methods = external,password,token,oauth1
    password = keystone.auth.plugins.password.Password
    token = keystone.auth.plugins.token.Token
    oauth1 = keystone.auth.plugins.oauth1.OAuth

    [token]
    driver = keystone.token.persistence.backends.memcache.Token
    provider = keystone.token.providers.uuid.Provider



Step 3: Update Keystone endpoints

Read port info from keystone.conf.

    # openstack endpoint list
    +----------------------------------+-----------+--------------+--------------+
    | ID                               | Region    | Service Name | Service Type |
    +----------------------------------+-----------+--------------+--------------+
    | 2b5851e0e39d420480a63626a7cffa82 | regionOne | glance       | image        |
    | 502c322cbb604ecc9929b0ff2e8a58b7 | regionOne | nova         | compute      |
    | d7dbfc2e5d904ca3be8571a5ad59a4d5 | regionOne | neutron      | network      |
    | 594f928ebde24a86b24dd095e64f240d | regionOne | keystone     | identity     |
    +----------------------------------+-----------+--------------+--------------+

    # openstack endpoint show keystone
    +--------------+----------------------------------------+
    | Field        | Value                                  |
    +--------------+----------------------------------------+
    | adminurl     | http://10.0.0.244:%(admin_port)s/v2.0  |
    | enabled      | True                                   |
    | id           | 594f928ebde24a86b24dd095e64f240d       |
    | internalurl  | http://10.0.0.244:%(public_port)s/v2.0 |
    | publicurl    | http://10.0.0.244:%(public_port)s/v2.0 |
    | region       | regionOne                              |
    | service_id   | 7088b1e64ba94ff180edf35971660f91       |
    | service_name | keystone                               |
    | service_type | identity                               |
    +--------------+----------------------------------------+





Step 4: Create admin and main code

Place following code as /var/www/keystone/admin and /var/www/keystone/main. Chown to keystone:apache.


    import logging
    import os

    from paste import deploy

    #from keystone.openstack.common import gettextutils
    ## NOTE(dstanek): gettextutils.enable_lazy() must be called before
    ## gettextutils._() is called to ensure it has the desired lazy #lookup behavior. This includes cases, like keystone.exceptions, #where gettextutils._() is called at import time.
    #gettextutils.enable_lazy()
    import oslo_i18n
    oslo_i18n.enable_lazy(enable=True)

    from keystone.common import dependency
    from keystone.common import environment
    from keystone.common import sql
    from keystone import config
    #from keystone.openstack.common import log
    from oslo_log import log
    #from keystone import service
    from keystone import backends


    CONF = config.CONF

    config.configure()
    sql.initialize()
    config.set_default_for_default_log_levels()

    CONF(project='keystone')
    config.setup_logging()

    environment.use_stdlib()
    name = os.path.basename(__file__)

    if CONF.debug:
        CONF.log_opt_values(log.getLogger(CONF.prog), logging.DEBUG)


    #drivers = service.load_backends()
    drivers = backends.load_backends()

    # NOTE(ldbragst): 'application' is required in this context by WSGI spec.
    # The following is a reference to Python Paste Deploy documentation
    # http://pythonpaste.org/deploy/
    application = deploy.loadapp('config:%s' % config.find_paste_config(),
                             name=name)

    dependency.resolve_future_dependencies()




Step 5: Restart Keystone


Restart httpd will also start keystone-all. No need to start openstack-keystone.service separately.            
   
    #systemctl restart httpd.service




