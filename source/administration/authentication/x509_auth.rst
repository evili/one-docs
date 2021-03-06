.. _x509_auth:

====================
x509 Authentication
====================

This guide will show you how to enable and use the x509 certificates authentication with OpenNebula. The x509 certificates can be used in two different ways in OpenNebula.

The first option that is explained in this guide enables us to use certificates with the CLI. In this case the user will generate a login token with his private key, OpenNebula will validate the certificate and decrypt the token to authenticate the user.

The second option enables us to use certificates with Sunstone and the Public Cloud servers included in OpenNebula. In this case the authentication is leveraged to Apache or any other SSL capable http proxy that has to be configured by the administrator. If this certificate is validated the server will encrypt those credentials using a server certificate and will send the token to OpenNebula.

Requirements
============

If you want to use the x509 certificates with Sunstone or one of the Public Clouds, you must deploy a SSL capable http proxy on top of them in order to handle the certificate validation.

Considerations & Limitations
============================

The X509 driver uses the certificate DN as user passwords. The x509 driver will remove any space in the certificate DN. This may cause problems in the unlikely situation that you are using a CA signing certificate subjects that only differ in spaces.

Configuration
=============

The following table summarizes the available options for the x509 driver (``/etc/one/auth/x509_auth.conf``):

+---------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| VARIABLE      | VALUE                                                                                                                                                                                                                                                                                          |
+===============+================================================================================================================================================================================================================================================================================================+
| :ca\_dir      | Path to the trusted CA directory. It should contain the trusted CA's for the server, each CA certificate shoud be name CA\_hash.0                                                                                                                                                              |
+---------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| :check\_crl   | By default, if you place CRL files in the CA directory in the form CA\_hash.r0, OpenNebula will check them. You can enforce CRL checking by defining :check\_crl, i.e. authentication will fail if no CRL file is found. You can always disable this feature by moving or renaming .r0 files   |
+---------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

Follow these steps to change oneadmin's authentication method to x509:

.. warning:: You should have another account in the oneadmin group, so you can revert these steps if the process fails.

-  :ref:`Change the oneadmin password <x509_auth_update_existing_users_to_x509_multiple_dn>` to the oneadmin certificate DN.

.. code::

    oneadmin@frontend $ oneuser chauth 0 x509 --x509 --cert /tmp/newcert.pem

-  :ref:`Add trusted CA certificates <x509_auth_add_and_remove_trusted_ca_certificates>` to the certificates directory

.. code::

    $ openssl x509 -noout -hash -in cacert.pem
    78d0bbd8

    $ sudo cp cacert.pem /etc/one/auth/certificates/78d0bbd8.0

-  :ref:`Create a login <x509_auth_user_login>` for oneadmin using the –x509 option. This token has a default expiration time set to 1 hour, you can change this value using the option –time.

.. code::

    oneadmin@frontend $ oneuser login oneadmin --x509 --cert newcert.pem --key newkey.pem
    Enter PEM pass phrase:
    export ONE_AUTH=/home/oneadmin/.one/one_x509

-  Set ONE\_AUTH to the x509 login file

.. code::

    oneadmin@frontend $ export ONE_AUTH=/home/oneadmin/.one/one_x509

Usage
=====

.. _x509_auth_add_and_remove_trusted_ca_certificates:

Add and Remove Trusted CA Certificates
--------------------------------------

You need to copy all trusted CA certificates to the certificates directory, renaming each of them as ``<CA_hash>.0``. The hash can be obtained with the openssl command:

.. code::

    $ openssl x509 -noout -hash -in cacert.pem
    78d0bbd8

    $ sudo cp cacert.pem /etc/one/auth/certificates/78d0bbd8.0

To stop trusting a CA, simply remove its certificate from the certificates directory.

This process can be done without restarting OpenNebula, the driver will look for the certificates each time an authentication request is made.

Create New Users
----------------

The users requesting a new account have to send their certificate, signed by a trusted CA, to the administrator. The following command will create a new user with username 'newuser', assuming that the user's certificate is saved in the file /tmp/newcert.pem:

.. code::

    oneadmin@frontend $ oneuser create newuser --x509 --cert /tmp/newcert.pem

This command will create a new user whose password contains the subject DN of his certificate. Therefore if the subject DN is known by the administrator the user can be created as follows:

.. code::

    oneadmin@frontend $ oneuser create newuser --x509 "user_subject_DN"

.. _x509_auth_update_existing_users_to_x509_multiple_dn:

Update Existing Users to x509 & Multiple DN
-------------------------------------------

You can change the authentication method of an existing user to x509 with the following command:

-  Using the user certificate:

.. code::

    oneadmin@frontend $ oneuser chauth <id|name> x509 --x509 --cert /tmp/newcert.pem

-  Using the user certificate subject DN:

.. code::

    oneadmin@frontend $ oneuser chauth <id|name> x509 --x509 "user_subject_DN"

You can also map multiple certificates to the same OpenNebula account. Just add each certificate DN separated with '\|' to the password field.

.. code::

    oneadmin@frontend $ oneuser passwd <id|name> --x509 "/DC=es/O=one/CN=user|/DC=us/O=two/CN=user"

.. _x509_auth_user_login:

User Login
----------

Users must execute the 'oneuser login' command to generate a login token. The token will be stored in the $ONE\_AUTH environment variable. The command requires the OpenNebula username, and the authentication method (``–x509`` in this case).

.. code::

    newuser@frontend $ oneuser login newuser --x509 --cert newcert.pem --key newkey.pem
    Enter PEM pass phrase:

The generated token has a default **expiration time** of 10 hours. You can change that with the ``–time`` option.

Tuning & Extending
==================

The x509 authentication method is just one of the drivers enabled in AUTH\_MAD. All drivers are located in ``/var/lib/one/remotes/auth``.

OpenNebula is configured to use x509 authentication by default. You can customize the enabled drivers in the AUTH\_MAD attribute of :ref:`oned.conf <oned_conf>`. More than one authentication method can be defined:

.. code::

    AUTH_MAD = [
        executable = "one_auth_mad",
        authn = "ssh,x509,ldap,server_cipher,server_x509"
    ]

Enabling x509 auth in Sunstone
==============================

Update the ``/etc/one/sunstone-server.conf`` :auth parameter to use the ``x509`` auth:

.. code::

        :auth: x509

