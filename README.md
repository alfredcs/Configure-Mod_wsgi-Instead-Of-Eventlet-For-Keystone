# Configure-Mod_wsgi-Instead-Of-Eventlet-For-Keystone
Starting with Kilo release, eventlet is replaced with mod_wsgi in front of Keystone service to avoid close_wait staus in TCP sessions. This page descibes wsgi and Keystone configurations in detail steps. Per Morgan Fainbery, the chief developer of Kerystone. The main reasons are that 
    "Keystone relies on apache/web-server modules to handle federated identity (validation of SAML, etc) and similar SSO type authentication (Kerberos)."
    "Eventlet has proven problematic when it comes to workloads within Keystone, notably that a number of actions cannot yield (either due to lacking in Eventlet, or that the dependent library uses C-bindings that eventlet is not able to work with)."
  
Followings configuration steps are for RedHat/CentOS platform. Please ajust accordingly for debian Linux platforms. 

Step 1: Create a /etc/httpd/conf.d/wsgi-keystone.conf

Step 2: Modify /etc/keystone/keystone.conf 

Step 3: Update Keystone endpoints

Step 4: Creat admin and main code

Step 5: Restart Keystone
    Restart httpd will also start keystone-all. No need to start openstack-keystone.service seperately. 
        systemctl restart httpd.service
