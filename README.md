osbs-master
===========

Role for deploying OpenShift master as part of OSBS - [OpenShift build
service](https://github.com/projectatomic/osbs-client/), service for building
layered Docker images.

It performs the necessary configuration OpenShift master, generates
configuration for nodes, and opens/closes OpenShift firewall port. It also
generates self-signed certificate that can be used by reverse proxy placed in
front of the builder.

The node configuration directories are tarred and copied to the ansible host
where the [osbs-node
role](https://github.com/projectatomic/ansible-role-osbs-node) can use them.

This role is part of
[ansible-osbs](https://github.com/projectatomic/ansible-osbs/) playbook for
deploying OpenShift build service. Please refer to that github repository for
[documentation](https://github.com/projectatomic/ansible-osbs/blob/master/README.md)
and [issue tracker](https://github.com/projectatomic/ansible-osbs/issues).

Role Variables
--------------

Expose the OpenShift port to the outside world? Set this to `false` when using
authenticating proxy on the localhost. Has no effect if `osbs_manage_firewalld`
is `false`.

    osbs_master_expose_port: true

Set to false if you don't use firewalld or do not want the playbook to modify
it.

    osbs_manage_firewalld: true

The `osbs_nodes` is a list of FQDNs of OpenShift nodes for which configuration
will be generated. It is suggested that the nodes are part of an ansible group
and the variable is set to be `group['your_node_group_name']`. By default it
contains only the master host, for easy one-host setup.

    osbs_nodes: ["{{ inventory_hostname }}"]

Configuration for nodes is generated on the master and then saven on the
ansible host to a directory where the osbs-node role can use them.

    osbs_node_config_dir: openshift-node-configs

If you are using authenticating proxy, this role can generate a self-signed
certificate that the proxy can use to authenticate itself to OpenShift. The
proxy needs the certificate and the key concatenated in one file
(`osbs_proxy_cert_file`). OpenShift needs to know the CA of the certificate,
which is configured in `osbs_proxy_ca_file` and which is the same as the
certificate because it is self-signed.

    osbs_proxy_cert_file: /etc/origin/proxy_selfsigned.crt
    osbs_proxy_key_file: /etc/origin/proxy_selfsigned.key
    osbs_proxy_certkey_file: /etc/httpd/openshift_proxy_certkey.crt
    osbs_proxy_ca_file: /etc/origin/proxy_selfsigned.crt

OpenShift authorization policy - which users should be assigned the view
(read-only), osbs-builder (read-write), and cluster-admin (admin) roles. In
default configuration, everyone has read/write access. The authentication is
handled by the proxy - if you are not using it the everyone connecting from the
outside belongs to the `system:unauthenticated` group.

Default setup:

    osbs_readonly_users: []
    osbs_readonly_groups: []
    osbs_readwrite_users: []
    osbs_readwrite_groups:
      - system:authenticated
      - system:unauthenticated
    osbs_admin_users: []
    osbs_admin_groups: []

Development with authenticating proxy:

    osbs_readonly_users: []
    osbs_readonly_groups: []
    osbs_readwrite_users: []
    osbs_readwrite_groups:
      - system:authenticated
    osbs_admin_users: []
    osbs_admin_groups: []

Example production configuration with only one user starting the builds:

    osbs_readonly_users: []
    osbs_readonly_groups:
      - system:authenticated
    osbs_readwrite_groups: []
    osbs_readwrite_users:
      - kojibuilder
    osbs_admin_users:
      - foo@EXAMPLE.COM
      - bar@EXAMPLE.COM
    osbs_admin_groups: []

Limit on the number of running pods. Undefine or set to -1 to remove limit.

    osbs_master_max_pods: 3

OpenShift master log level (0-5).

    osbs_master_log_level: 3

Changing the auth provider.

    # Specify different identity providers and options needed for the master-config
    # template
    #
    # Currently supported options are:
    #   request_header
    #   htpasswd_provider
    osbs_identity_provider: "request_header"

Options for the request_header auth type.

    osbs_identity_request:
      name: request_header
      challenge: true
      login: true

Options for the htpasswd_provider auth type. NOTE: You will need to create an
htpasswd file and place it in the location of `provider_file` either by hand or
via an ansible task for hosts that require it.

    osbs_identity_htpasswd:
      name: htpasswd_provider
      challenge: true
      login: true
      provider_file: /etc/openshift/htpasswd

[Image garbage
collection](https://docs.openshift.org/latest/admin_guide/garbage_collection.html#image-garbage-collection)
can be configured with following variables:

    osbs_image_gc_high_threshold: 90
    osbs_image_gc_low_threshold: 80

Deploy yum proxy from a docker image (disabled by default). Variable
`osbs_yum_proxy_name` sets the name of the deploymentconfig/service OpenShift
objects.

    osbs_yum_proxy_image: docker.io/vrutkovs/docker-squid
    osbs_yum_proxy_name: yum-proxy

Dependencies
------------

Docker is expected to be installed and configured (e.g. storage, insecure
registries) on the remote host.

OpenShift is expected to be installed on the remote host. This can be
accomplished by the
[install-openshift](https://github.com/projectatomic/ansible-role-install-openshift)
role.

Example Playbook
----------------

Simple development deployment:

    - hosts: builders
      roles:
        - install-openshift
        - osbs-master
        - osbs-node
        - atomic-reactor

Deployment behind authentication proxy that only allows the *kojibuilder* user
to start builds (and everyone to view them).

    - hosts: builders
      roles:
        - install-openshift
        - role: osbs-master
          osbs_master_expose_port: false
          osbs_readonly_users: []
          osbs_readonly_groups:
            - system:authenticated
            - system:unauthenticated
          osbs_readwrite_groups: []
          osbs_readwrite_users:
            - kojibuilder
          osbs_admin_users: []
          osbs_admin_groups: []
        - osbs-node
        - atomic-reactor
        - role: osbs-proxy
          osbs_proxy_type: kerberos
          osbs_proxy_kerberos_keytab_file: /etc/HTTP-FQDN.EXAMPLE.COM.keytab
          osbs_proxy_kerberos_realm: EXAMPLE.COM
          osbs_proxy_ssl_cert_file: /etc/fqdn.example.com.crt
          osbs_proxy_ssl_key_file: /etc/fqdn.example.com.key
          osbs_proxy_ip_whitelist:
            - subnet: 192.168.66.0/24
              user: kojibuilder

Simple deployment with multiple nodes:

    - hosts: all
      roles:
        - install-openshift

    - hosts: masters
      roles:
        - role: osbs-master
          osbs_nodes: "{{ groups['nodes'] }}"

    - hosts: nodes
      roles:
        - osbs-node
        - atomic-reactor

License
-------

BSD

Author Information
------------------

Martin Milata &lt;mmilata@redhat.com&gt;
