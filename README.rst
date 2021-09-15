nova-vnc-cert
==============

This is an ansible playbook to deploy vnc certificates and keys to be used by
libvirt and nova-novncproxy in TACO platform.

Its purpose is to prevent VNC console connection from clients 
without certificate or with invalid certificate.


Installation on an existing TACO cluster
-------------------------------------------

Deploy vnc certificate secrets
+++++++++++++++++++++++++++++++++

Run nova-vnc-cert ansible playbook. It will create vnc-server-secret and
vnc-client-secret.

	$ ansible-playbook site.yml

Check two secrets are created.::

	$ kubectl get secrets vnc-{server,client}-secret -n openstack
    NAME                TYPE     DATA   AGE
    vnc-server-secret   Opaque   3      21h
    vnc-client-secret   Opaque   3      21h

Put the libvirt image in the register
+++++++++++++++++++++++++++++++++++++++

The libvirt image with gnutls can be downloaded at harbor
(192.168.151.110/burrito/cloudpc-libvirt:train-gnutls) in CICD zone

Get the image tarball and push into the site's registry.


Edit openstack-manifest.yaml
+++++++++++++++++++++++++++++++

taco/inventory/<sitename>/openstack-manifest.yaml

All plus sign(+) is the added lines and minus sign(-) is the deleted or
comment-out lines.

Go to libvirt section and change libvirt image to use gnutls support.::

    values:
      release_group: null
      images:
        pull_policy: IfNotPresent
        tags:
    -      #libvirt: 192.168.21.40:5000/burrito/centos-source-nova-cloudpc-libvirt:taco-train-skb-1.1.0.9
    +      libvirt: 192.168.21.40:5000/burrito/cloudpc-libvirt:train-gnutls

Add vnc_tls qemu configuration in libvirt section.::

  	conf:
  	...
  	  qemu:
  	  ...
    +     vnc_tls: 1
    +     vnc_tls_x509_cert_dir: "/etc/pki/libvirt-vnc"
    +     vnc_tls_x509_verify: 1
          spice_tls: 1
          spice_tls_x509_cert_dir: "/etc/pki/libvirt-spice"
          seccomp_sandbox: 1
          spice_listen: 0.0.0.0

Add libvirt-vnc mounts in libvirt section.::

          spice_listen: 0.0.0.0
    +  pod:
    +    mounts:
    +      libvirt:
    +        libvirt:
    +          volumeMounts:
    +          - name: vnc-server-secret
    +            mountPath: /etc/pki/libvirt-vnc
    +            readOnly: true
    +          volumes:
    +          - name: vnc-server-secret
    +            secret:
    +              secretName: vnc-server-secret
    +              items:
    +              - key: ca_crt
    +                path: ca-cert.pem
    +              - key: server_crt
    +                path: server-cert.pem
    +              - key: server_key
    +                path: server-key.pem
    source:
    type: local


Go to nova section and add vnc configuration.::


        notifications:
          bdms_in_notifications: true
        vnc:
    +     auth_scemes: vencrypt
    +     vencrypt_client_key: /etc/pki/nova-novncproxy/client-key.pem
    +     vencrypt_client_cert: /etc/pki/nova-novncproxy/client-cert.pem
    +     vencrypt_ca_certs: /etc/pki/nova-novncproxy/ca-cert.pem
          novncproxy_base_url: http://192.168.21.40:30608/vnc_auto.html
        libvirt:
          connection_uri: "qemu+tcp://127.0.0.1/system"
          images_type: qcow2

Add nova-novncproxy mounts.::

      network:
        port:
          api:
            public: 8080
    pod:
    +  mounts:
    +    nova_novncproxy:
    +      nova_novncproxy:
    +        volumeMounts:
    +        - name: vnc-client-secret
    +          mountPath: /etc/pki/nova-novncproxy
    +          readOnly: true
    +        volumes:
    +        - name: vnc-client-secret
    +          secret:
    +            secretName: vnc-client-secret
    +            items:
    +            - key: ca_crt
    +              path: ca-cert.pem
    +            - key: client_crt
    +              path: client-cert.pem
    +            - key: client_key
    +              path: client-key.pem
      security_context:
        nova:

Run armada
+++++++++++++

Run armada to apply the changed manifest.::

	$ cd ~/taco
	$ sudo docker exec -u root armada armada apply --tiller-host <controller_ip> --tiller-port 32134 --timeout 600 ~/taco/inventory/<site_name>/openstack-manifest.yaml

See if nova-novncproxy and libvirt pods are running.


Check 
------

Now VNC console connection is allowed only to the client with valid 
certificate. The nova-novncproxy is set up to have a valid client certificate.
So you can connect to VNC console using horizon dashboard.

There is one thing I should mention.::

    VM should be stopped and started if it was running before 
    nova-vnc-cert setup because it did not load the certificates
    when it was launched.
	

Let's check the VNC console connection.

#. Go to horizon and login as admin.
#. Go to the instance 
#. Select one instance and click console tab.
#. You will see the instance console.

Now use vnc client like tightvnc viewer and connect to VNC console of VM.
It will be rejected since it does not have the valid client certificate.


