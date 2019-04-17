=========
Checklist
=========

.. _check_image_01:

Check-Image-01: Is user/group ownership of config files set to root/glance?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configuration files contain critical parameters and information required for
smooth functioning of the component. If an unprivileged user, either
intentionally or accidentally, modifies or deletes any of the parameters or
the file itself then it would cause severe availability issues resulting in a
denial of service to the other end users. Therefore, user ownership of such
critical configuration files must be set to ``root`` and group ownership
must be set to ``glance``. Additionally, the containing directory should have
the same ownership to ensure that new files are owned correctly.

Run the following commands:

.. code:: console

   $ stat -L -c "%U %G" /etc/glance/glance-api-paste.ini | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/glance-api.conf | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/glance-cache.conf | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/glance-manage.conf | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/glance-registry-paste.ini | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/glance-registry.conf | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/glance-scrubber.conf | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/glance-swift-store.conf | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/policy.json | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/schema-image.json | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance/schema.json | egrep "root glance"
   $ stat -L -c "%U %G" /etc/glance | egrep "root glance"

**Pass:** If user and group ownership of all these configuration files is set
to root and glance respectively. The above commands show output of root glance.

**Fail:** If the above commands do not return any output.

.. _check_image_02:

Check-Image-02: Are strict permissions set for configuration files?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Similar to the previous check, we recommend you set strict access
permissions for such configuration files.

Run the following commands:

.. code:: console

    $ stat -L -c "%a" /etc/glance/glance-api-paste.ini
    $ stat -L -c "%a" /etc/glance/glance-api.conf
    $ stat -L -c "%a" /etc/glance/glance-cache.conf
    $ stat -L -c "%a" /etc/glance/glance-manage.conf
    $ stat -L -c "%a" /etc/glance/glance-registry-paste.ini
    $ stat -L -c "%a" /etc/glance/glance-registry.conf
    $ stat -L -c "%a" /etc/glance/glance-scrubber.conf
    $ stat -L -c "%a" /etc/glance/glance-swift-store.conf
    $ stat -L -c "%a" /etc/glance/policy.json
    $ stat -L -c "%a" /etc/glance/schema-image.json
    $ stat -L -c "%a" /etc/glance/schema.json
    $ stat -L -c "%a" /etc/glance

A broader restriction is also possible: if the containing directory is set
to 750, the guarantee is made that newly created files inside this directory
would have the desired permissions.

**Pass:** If permissions are set to 640 or stricter, or the containing
directory is set to 750. The permissions of 640/750 translates into owner r/w,
group r, and no rights to others. For example, ``u=rw,g=r,o=``.

.. note::

   With :ref:`check_image_01`, and permissions set to 640, root has
   read/write access and glance has read access to these configuration files. The
   access rights can also be validated using the following command. This command
   will only be available on your system if it supports ACLs.

.. code:: console

    $ getfacl --tabular -a /etc/glance/glance-api.conf
    getfacl: Removing leading '/' from absolute path names
    # file: /etc/glance/glance-api.conf
    USER   root  rw-
    GROUP  glance  r--
    mask         r--
    other        ---

**Fail:** If permissions are not set to at least 640.

.. _check_image_03:

Check-Image-03: Is keystone used for authentication?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

OpenStack supports various authentication strategies including noauth, and
keystone. If the ``noauth`` strategy is used, then users can interact with
OpenStack services without any authentication. This could be a potential
risk since an attacker might gain unauthorized access to the OpenStack
components. We strongly recommend that all services must be authenticated
with keystone using their service accounts.

**Pass:** If value of parameter ``auth_strategy`` under ``[DEFAULT]`` section
in ``/etc/glance/glance-api.conf`` is set to ``keystone`` and value of
parameter ``auth_strategy`` under ``[DEFAULT]`` section in ``/etc/glance
/glance-registry.conf`` is set to ``keystone``.

**Fail:** If value of parameter ``auth_strategy`` under ``[DEFAULT]`` section
in ``/etc/glance/glance-api.conf`` is set to ``noauth`` or value of parameter
``auth_strategy`` under ``[DEFAULT]`` section in ``/etc/glance/glance-
registry.conf`` is set to ``noauth``.

.. _check_image_04:

Check-Image-04: Is TLS enabled for authentication?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

OpenStack components communicate with each other using various protocols and
the communication might involve sensitive or confidential data. An attacker
may try to eavesdrop on the channel in order to get access to sensitive
information. All components must communicate with each other using a
secured communication protocol.

**Pass:** If value of parameter ``www_authenticate_uri`` under
``[keystone_authtoken]`` section in ``/etc/glance/glance-api.conf`` is set to
the Identity API endpoint starting with ``https://``, and the value of the
parameter ``insecure`` is under the same ``[keystone_authtoken]`` section
in the same ``/etc/glance/glance-registry.conf`` is set to ``False``.

**Fail:** If value of parameter ``www_authenticate_uri`` under
``[keystone_authtoken]`` section in ``/etc/glance/glance-api.conf`` is not set
to Identity API endpoint starting with ``https://``, or value of parameter
``insecure`` under the same ``[keystone_authtoken]`` section in the same
``/etc/glance/glance-api.conf`` is set to ``True``.

.. _check_image_05:

Check-Image-05: Are masked port scans prevented?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``copy_from`` feature in Image Service API v1 supplied by Glance can allow
an attacker to perform masked network port scans. If the v1 API is enabled,
this policy should be set to a restricted value.

**Pass:** If value of parameter ``copy_from`` in ``/etc/glance/policy.json``
is set to a restricted value, for example ``role:admin``.

**Fail:** If value of parameter ``copy_from`` in ``/etc/glance/policy.json``
is not set.
