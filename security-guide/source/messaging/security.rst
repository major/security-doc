==================
Messaging security
==================

This section discusses security hardening approaches for the three most
common message queuing solutions used in OpenStack: RabbitMQ, Qpid, and
ZeroMQ.

Messaging transport security
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

AMQP based solutions (Qpid and RabbitMQ) support transport-level
security using TLS. ZeroMQ messaging does not natively support TLS, but
transport-level security is possible using labelled IPsec or CIPSO
network labels.

We highly recommend enabling transport-level cryptography for your
message queue. Using TLS for the messaging client connections provides
protection of the communications from tampering and eavesdropping
in-transit to the messaging server. Below is guidance on how TLS is
typically configured for the two popular messaging servers Qpid and
RabbitMQ. When configuring the trusted certificate authority (CA) bundle
that your messaging server uses to verify client connections, it is
recommended that this be limited to only the CA used for your nodes,
preferably an internally managed CA. The bundle of trusted CAs will
determine which client certificates will be authorized and pass the
client-server verification step of the setting up the TLS connection.
Note, when installing the certificate and key files, ensure that the
file permissions are restricted, for example using ``chmod 0600``, and
the ownership is restricted to the messaging server daemon user to
prevent unauthorized access by other processes and users on the
messaging server.

RabbitMQ server SSL configuration
---------------------------------

The following lines should be added to the system-wide RabbitMQ
configuration file, typically :file:`/etc/rabbitmq/rabbitmq.config`:

::

    [
      {rabbit, [
         {tcp_listeners, [] },
         {ssl_listeners, [{"<IP address or hostname of management network interface>", 5671}] },
         {ssl_options, [{cacertfile,"/etc/ssl/cacert.pem"},
                        {certfile,"/etc/ssl/rabbit-server-cert.pem"},
                        {keyfile,"/etc/ssl/rabbit-server-key.pem"},
                        {verify,verify_peer},
                        {fail_if_no_peer_cert,true}]}
       ]}
    ].

Note, the ``tcp_listeners`` option is set to ``[]`` to prevent it from
listening an on non-SSL port. The ``ssl_listeners`` option should be
restricted to only listen on the management network for the services.

For more information on RabbitMQ SSL configuration see:

-  `RabbitMQ Configuration <http://www.rabbitmq.com/configure.html>`__

-  `RabbitMQ SSL <http://www.rabbitmq.com/ssl.html>`__

Qpid server SSL configuration
-----------------------------

The Apache Foundation has a messaging security guide for Qpid. See:

-  `Apache Qpid
   SSL <http://qpid.apache.org/releases/qpid-0.32/cpp-broker/book/chap-Messaging_User_Guide-Security.html#sect-Messaging_User_Guide-Security-Encryption_using_SSL>`__

.. _queue-authentication-and-access-control:

Queue authentication and access control
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

RabbitMQ and Qpid offer authentication and access control mechanisms for
controlling access to queues. ZeroMQ offers no such mechanisms.

Simple Authentication and Security Layer (SASL) is a framework for
authentication and data security in Internet protocols. Both RabbitMQ
and Qpid offer SASL and other pluggable authentication mechanisms beyond
simple user names and passwords that allow for increased authentication
security. While RabbitMQ supports SASL, support in OpenStack does not
currently allow for requesting a specific SASL authentication mechanism.
RabbitMQ support in OpenStack allows for either user name and password
authentication over an unencrypted connection or user name and password
in conjunction with X.509 client certificates to establish the secure
TLS connection.

We recommend configuring X.509 client certificates on all the OpenStack
service nodes for client connections to the messaging queue and where
possible (currently only Qpid) perform authentication with X.509 client
certificates. When using user names and passwords, accounts should be
created per-service and node for finer grained auditability of access to
the queue.

Before deployment, consider the TLS libraries that the queuing servers
use. Qpid uses Mozilla's NSS library, whereas RabbitMQ uses Erlang's TLS
module which uses OpenSSL.

Authentication configuration example: RabbitMQ
----------------------------------------------

On the RabbitMQ server, delete the default ``guest`` user:

.. code:: console

    # rabbitmqctl delete_user guest

On the RabbitMQ server, for each OpenStack service or node that
communicates with the message queue set up user accounts and privileges:

.. code:: console

    # rabbitmqctl add_user compute01 RABBIT_PASS
    # rabbitmqctl set_permissions compute01 ".*" ".*" ".*"

Replace RABBIT\_PASS with a suitable password.

For additional configuration information see:

-  `RabbitMQ Access
   Control <http://www.rabbitmq.com/access-control.html>`__

-  `RabbitMQ
   Authentication <http://www.rabbitmq.com/authentication.html>`__

-  `RabbitMQ Plugins <http://www.rabbitmq.com/plugins.html>`__

-  `RabbitMQ SASL External
   Auth <http://hg.rabbitmq.com/rabbitmq-auth-mechanism-ssl/file/rabbitmq_v3_1_3/README>`__

OpenStack service configuration: RabbitMQ
-----------------------------------------

.. code:: ini

    [DEFAULT]
    rpc_backend = nova.openstack.common.rpc.impl_kombu
    rabbit_use_ssl = True
    rabbit_host = RABBIT_HOST
    rabbit_port = 5671
    rabbit_user = compute01
    rabbit_password = RABBIT_PASS
    kombu_ssl_keyfile = /etc/ssl/node-key.pem
    kombu_ssl_certfile = /etc/ssl/node-cert.pem
    kombu_ssl_ca_certs = /etc/ssl/cacert.pem

Authentication configuration example: Qpid
------------------------------------------

For configuration information see:

-  `Apache Qpid
   Authentication <http://qpid.apache.org/releases/qpid-0.32/cpp-broker/book/chap-Messaging_User_Guide-Security.html#sect-Messaging_User_Guide-Security-User_Authentication>`__

-  `Apache Qpid
   Authorization <http://qpid.apache.org/releases/qpid-0.32/cpp-broker/book/chap-Messaging_User_Guide-Security.html#sect-Messaging_User_Guide-Security-Authorization>`__

OpenStack service configuration: Qpid
-------------------------------------

.. code:: ini

    [DEFAULT]
    rpc_backend = nova.openstack.common.rpc.impl_qpid
    qpid_protocol = ssl
    qpid_hostname = <IP or hostname of management network interface of messaging server>
    qpid_port = 5671
    qpid_username = compute01
    qpid_password = QPID_PASS

Optionally, if using SASL with Qpid specify the SASL mechanisms in use
by adding:

.. code:: ini

    qpid_sasl_mechanisms = <space separated list of SASL mechanisms to use for auth>

Message queue process isolation and policy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each project provides a number of services which send and consume
messages. Each binary which sends a message is expected to consume
messages, if only replies, from the queue.

Message queue service processes should be isolated from each other and
other processes on a machine.

Namespaces
----------

Network namespaces are highly recommended for all services running on
OpenStack Compute Hypervisors. This will help prevent against the
bridging of network traffic between VM guests and the management
network.

When using ZeroMQ messaging, each host must run at least one ZeroMQ
message receiver to receive messages from the network and forward
messages to local processes through IPC. It is possible and advisable to
run an independent message receiver per project within an IPC namespace,
along with other services within the same project.

Network policy
--------------

Queue servers should only accept connections from the management
network. This applies to all implementations. This should be implemented
through configuration of services and optionally enforced through global
network policy.

When using ZeroMQ messaging, each project should run a separate ZeroMQ
receiver process on a port dedicated to services belonging to that
project. This is equivalent to the AMQP concept of control exchanges.

Mandatory access controls
-------------------------

Use both mandatory access controls (MACs) and discretionary access
controls (DACs) to restrict the configuration for processes to only
those processes. This restriction prevents these processes from being
isolated from other processes that run on the same machine(s).
