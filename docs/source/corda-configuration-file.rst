Node configuration
==================

.. contents::

File location
-------------
When starting a node, the ``corda.jar`` file defaults to reading the node's configuration from a ``node.conf`` file in
the directory from which the command to launch Corda is executed. There are two command-line options to override this
behaviour:

* The ``--config-file`` command line option allows you to specify a configuration file with a different name, or at
  different file location. Paths are relative to the current working directory

* The ``--base-directory`` command line option allows you to specify the node's workspace location. A ``node.conf``
  configuration file is then expected in the root of this workspace

If you specify both command line arguments at the same time, the node will fail to start.

Format
------
The Corda configuration file uses the HOCON format which is superset of JSON. Please visit
`<https://github.com/typesafehub/config/blob/master/HOCON.md>`_ for further details.

Defaults
--------
A set of default configuration options are loaded from the built-in resource file ``/node/src/main/resources/reference.conf``.
This file can be found in the ``:node`` gradle module of the `Corda repository <https://github.com/corda/corda>`_. Any
options you do not specify in your own ``node.conf`` file will use these defaults.

Here are the contents of the ``reference.conf`` file:

.. literalinclude:: ../../node/src/main/resources/reference.conf
    :language: javascript

Fields
------
The available config fields are listed below. ``baseDirectory`` is available as a substitution value and contains the
absolute path to the node's base directory.

:myLegalName: The legal identity of the node. This acts as a human-readable alias to the node's public key and can be used with
    the network map to look up the node's info. This is the name that is used in the node's certificates (either when requesting them
    from the doorman, or when auto-generating them in dev mode). At runtime, Corda checks whether this name matches the
    name in the node's certificates.

:keyStorePassword: The password to unlock the KeyStore file (``<workspace>/certificates/sslkeystore.jks``) containing the
    node certificate and private key.

    .. note:: This is the non-secret value for the development certificates automatically generated during the first node run.
        Longer term these keys will be managed in secure hardware devices.

:trustStorePassword: The password to unlock the Trust store file (``<workspace>/certificates/truststore.jks``) containing
    the Corda network root certificate. This is the non-secret value for the development certificates automatically
    generated during the first node run.

    .. note:: Longer term these keys will be managed in secure hardware devices.

:database: Database configuration:

        :serverNameTablePrefix: Prefix string to apply to all the database tables. The default is no prefix.
        :transactionIsolationLevel: Transaction isolation level as defined by the ``TRANSACTION_`` constants in
            ``java.sql.Connection``, but without the "TRANSACTION_" prefix. Defaults to REPEATABLE_READ.
        :exportHibernateJMXStatistics: Whether to export Hibernate JMX statistics (caution: expensive run-time overhead)

:dataSourceProperties: This section is used to configure the jdbc connection and database driver used for the nodes persistence.
    Currently the defaults in ``/node/src/main/resources/reference.conf`` are as shown in the first example. This is currently
    the only configuration that has been tested, although in the future full support for other storage layers will be validated.

:messagingServerAddress: The address of the ArtemisMQ broker instance. If not provided the node will run one locally.

:p2pAddress: The host and port on which the node is available for protocol operations over ArtemisMQ.

    .. note:: In practice the ArtemisMQ messaging services bind to all local addresses on the specified port. However,
        note that the host is the included as the advertised entry in the NetworkMapService. As a result the value listed
        here must be externally accessible when running nodes across a cluster of machines. If the provided host is unreachable,
        the node will try to auto-discover its public one.

:rpcAddress: The address of the RPC system on which RPC requests can be made to the node. If not provided then the node will run without RPC. This is now deprecated in favour of the ``rpcSettings`` block.

:rpcSettings: Options for the RPC server.

        :useSsl: (optional) boolean, indicates whether the node should require clients to use SSL for RPC connections, defaulted to ``false``.
        :standAloneBroker: (optional) boolean, indicates whether the node will connect to a standalone broker for RPC, defaulted to ``false``.
        :address: (optional) host and port for the RPC server binding, if any.
        :adminAddress: (optional) host and port for the RPC admin binding (only required when ``useSsl`` is ``false``, because the node connects to Artemis using SSL to ensure admin privileges are not accessible outside the node).
        :ssl: (optional) SSL settings for the RPC server.

                :keyStorePassword: password for the key store.
                :trustStorePassword: password for the trust store.
                :certificatesDirectory: directory in which the stores will be searched, unless absolute paths are provided.
                :sslKeystore: absolute path to the ssl key store, defaulted to ``certificatesDirectory / "sslkeystore.jks"``.
                :trustStoreFile: absolute path to the trust store, defaulted to ``certificatesDirectory / "truststore.jks"``.

:security: Contains various nested fields controlling user authentication/authorization, in particular for RPC accesses. See
    :doc:`clientrpc` for details.

:webAddress: The host and port on which the webserver will listen if it is started. This is not used by the node itself.

    .. note:: If HTTPS is enabled then the browser security checks will require that the accessing url host name is one
        of either the machine name, fully qualified machine name, or server IP address to line up with the Subject Alternative
        Names contained within the development certificates. This is addition to requiring the ``/config/dev/corda_dev_ca.cer``
        root certificate be installed as a Trusted CA.

    .. note:: The driver will not automatically create a webserver instance, but the Cordformation will. If this field
              is present the web server will start.

:notary: Optional configuration object which if present configures the node to run as a notary. If part of a Raft or BFT SMaRt
    cluster then specify ``raft`` or ``bftSMaRt`` respectively as described below. If a single node notary then omit both.

    :validating: Boolean to determine whether the notary is a validating or non-validating one.

    :raft: If part of a distributed Raft cluster specify this config object, with the following settings:

        :nodeAddress: The host and port to which to bind the embedded Raft server. Note that the Raft cluster uses a
            separate transport layer for communication that does not integrate with ArtemisMQ messaging services.

        :clusterAddresses: Must list the addresses of all the members in the cluster. At least one of the members must
            be active and be able to communicate with the cluster leader for the node to join the cluster. If empty, a
            new cluster will be bootstrapped.

    :bftSMaRt: If part of a distributed BFT-SMaRt cluster specify this config object, with the following settings:

        :replicaId: The zero-based index of the current replica. All replicas must specify a unique replica id.

        :clusterAddresses: Must list the addresses of all the members in the cluster. At least one of the members must
            be active and be able to communicate with the cluster leader for the node to join the cluster. If empty, a
            new cluster will be bootstrapped.

    :custom: If `true`, will load and install a notary service from a CorDapp. See :doc:`tutorial-custom-notary`.

    Only one of ``raft``, ``bftSMaRt`` or ``custom`` configuration values may be specified.

:rpcUsers: A list of users who are authorised to access the RPC system. Each user in the list is a config object with the
    following fields:

    :username: Username consisting only of word characters (a-z, A-Z, 0-9 and _)
    :password: The password
    :permissions: A list of permissions for starting flows via RPC. To give the user the permission to start the flow
        ``foo.bar.FlowClass``, add the string ``StartFlow.foo.bar.FlowClass`` to the list. If the list
        contains the string ``ALL``, the user can start any flow via RPC. This value is intended for administrator
        users and for development.

:devMode: This flag sets the node to run in development mode. On startup, if the keystore ``<workspace>/certificates/sslkeystore.jks``
    does not exist, a developer keystore will be used if ``devMode`` is true. The node will exit if ``devMode`` is false
    and the keystore does not exist. ``devMode`` also turns on background checking of flow checkpoints to shake out any
    bugs in the checkpointing process. Also, if ``devMode`` is true, Hibernate will try to automatically create the schema required by Corda
    or update an existing schema in the SQL database; if ``devMode`` is false, Hibernate will simply validate an existing schema
    failing on node start if this schema is either not present or not compatible.

:detectPublicIp: This flag toggles the auto IP detection behaviour, it is enabled by default. On startup the node will
    attempt to discover its externally visible IP address first by looking for any public addresses on its network
    interfaces, and then by sending an IP discovery request to the network map service. Set to ``false`` to disable.

:compatibilityZoneURL: The root address of Corda compatibility zone network management services, it is used by the Corda node to register with the network and
    obtain Corda node certificate, (See :doc:`permissioning` for more information.) and also used by the node to obtain network map information.

:jvmArgs: An optional list of JVM args, as strings, which replace those inherited from the command line when launching via ``corda.jar``
    only. e.g. ``jvmArgs = [ "-Xmx220m", "-Xms220m", "-XX:+UseG1GC" ]``

:systemProperties: An optional map of additional system properties to be set when launching via ``corda.jar`` only.  Keys and values
    of the map should be strings. e.g. ``systemProperties = { visualvm.display.name = FooBar }``

:jarDirs: An optional list of file system directories containing JARs to include in the classpath when launching via ``corda.jar`` only.
    Each should be a string.  Only the JARs in the directories are added, not the directories themselves.  This is useful
    for including JDBC drivers and the like. e.g. ``jarDirs = [ 'lib' ]``

:sshd: If provided, node will start internal SSH server which will provide a management shell. It uses the same credentials and permissions as RPC subsystem. It has one required parameter.

    :port: The port to start SSH server on

:jmxMonitoringHttpPort: If set, will enable JMX metrics reporting via the Jolokia HTTP/JSON agent on the corresponding port.
    Default Jolokia access url is http://127.0.0.1:port/jolokia/

:transactionCacheSizeMegaBytes: Optionally specify how much memory should be used for caching of ledger transactions in memory.
            Otherwise defaults to 8MB plus 5% of all heap memory above 300MB.

:attachmentContentCacheSizeMegaBytes: Optionally specify how much memory should be used to cache attachment contents in memory.
            Otherwise defaults to 10MB

:attachmentCacheBound: Optionally specify how many attachments should be cached locally. Note that this includes only the key and
            metadata, the content is cached separately and can be loaded lazily. Defaults to 1024.

Examples
--------

General node configuration file for hosting the IRSDemo services:

.. literalinclude:: example-code/src/main/resources/example-node.conf
   :language: javascript

Simple notary configuration file:

.. parsed-literal::

    myLegalName : "O=Notary Service,OU=corda,L=London,C=GB"
    keyStorePassword : "cordacadevpass"
    trustStorePassword : "trustpass"
    p2pAddress : "localhost:12345"
    rpcSettings = {
        useSsl = false
        standAloneBroker = false
        address : "my-corda-node:10003"
        adminAddress : "my-corda-node:10004"
    }
    webAddress : "localhost:12347"
    notary : {
        validating : false
    }
    devMode : true
    compatibilityZoneURL : "https://cz.corda.net"