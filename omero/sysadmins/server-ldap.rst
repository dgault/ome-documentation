LDAP authentication
===================

`LDAP <https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol>`_
is an open standard for querying and modifying directory services that
is commonly used for authentication, authorization and accounting (AAA).
OMERO.server supports the use of an LDAP server to query (but not
modify) AAA information for the purposes of automatic user creation.

This allows OMERO users to be automatically created and placed in groups
according to your existing institution policies. This can significantly
simplify your user administration burden. Note that OMERO has its own
concept of "groups" that is quite distinct from LDAP groups.

The OMERO.server LDAP implementation can handle a number of use cases.
For example:

- Allow every "inetOrgPerson" under :property:`omero.ldap.base` to login
- but restrict access based upon an arbitrary LDAP filter, e.g.

::

    omero.ldap.user_filter=(memberOf=cn=someGoup,ou=Lab,o=College)

- and add that user to some number of groups, e.g.

::

    omero.ldap.new_user_group=:query:(member=@{dn})

How it works
------------

On login, the username provided is searched for in OMERO. If the name
does not exist, then the LDAP plugin is queried for a username matching
the system-wide user filter. If such an LDAP entry exists and the
password matches, a new user with the given username is created, and the
user is added to any groups which match the ``new_user_group`` setting.

On subsequent logins, the user filter and the password are again checked
against the LDAP server, and if there is no longer a match, login is refused.
If you would prefer to only have the ``user_filter`` applied during user
creation and not on every login, see :ref:`legacy_password_providers`.

You can take existing non-LDAP users and 'upgrade' them to using LDAP with the
OMERO command line tool, see :ref:`ldap_setdn`. You can also use
:program:`omero ldap create` to add an ldap user to OMERO groups without requiring
them to log in first, see :doc:`cli/usergroup` for details.

LDAP properties
---------------

The LDAP plugin is configured via several configuration properties, all
starting with ``omero.ldap`` (see :ref:`ldap_configuration`).

Some of the property values are passed directly to the underlying LDAP library
(`Spring LDAP <http://projects.spring.io/spring-ldap/>`_), which in turn makes
use of the Java API. OMERO does not modify the error messages thrown by the
library or by Java, so please consult the appropriate documentation to
diagnose any low-level problems.

.. note::

    Please remember that once a change has been made, a server restart will be
    needed.

Minimum configuration
^^^^^^^^^^^^^^^^^^^^^

The following properties are the minimum requirements for logging in to OMERO
using LDAP.

::

    omero.ldap.config=true
    omero.ldap.urls=ldap://localhost:389
    omero.ldap.username=
    omero.ldap.password=
    omero.ldap.base=ou=example,o=com

After having configured your connection, you can turn LDAP on and off
between restarts by setting :property:`omero.ldap.config` to false. The ``base``
property determines where in the LDAP tree searches will begin. No users or
groups will be found if they are not under the base provided.

User lookup
^^^^^^^^^^^

Two user properties are used to look up users by login name and,
if necessary, create new users based on the information in LDAP.

::

    omero.ldap.user_filter=(objectClass=person)
    omero.ldap.user_mapping=omeName=cn,firstName=givenName,lastName=sn,email=mail,institution=department,middleName=middleName

:property:`omero.ldap.user_filter` will be AND'ed to the username query, and can
contain any valid LDAP filter string. The username query is taken from
the LDAP attribute which gets mapped to "omeName" by
:property:`omero.ldap.user_mapping`. Here, the "cn" is mapped to "omeName", so the
username query is ``(cn=[login name])``. The final query is
``(&(objectClass=person)(cn=[login name]))``, which must return a single
result to be considered valid.

Group lookup
^^^^^^^^^^^^

Three group properties are all concerned with what groups a user
will be placed in on creation.

::

    omero.ldap.group_filter=(objectClass=groupOfNames)
    omero.ldap.group_mapping=name=cn
    omero.ldap.new_user_group=default

The group filter and group mapping work
just as the user filter and mapping do, in that the group name query
will be AND'd with the ``group_filter``. In this case, the final query
would be ``(&(objectClass=groupOfNames)(cn=[group name]))``. However,
these properties may not be used depending on
the value of ``new_user_group``, which can have several different values:

- If not prefixed at all, then the value is simply the name of a group which
  all users from LDAP should be added to.
- If prefixed with ``:ou:``, then a user's last organizational unit (OU) will
  be used as his or her group. For example, the user with the DN
  "cn=frank,ou=TheLab,ou=LifeSciences,o=TheCollege" will be placed in the
  group "TheLab".
- If prefixed with ``:attribute:``, then the rest of the string is taken to be
  an attribute all of whose values will be taken as group names. For example,
  ``omero.ldap.new_user_group=:attribute:memberOf`` would add a user to all
  the groups named by memberOf. You can prefix this value with ``filtered_``
  to have the ``group_filter`` applied to the attribute values, i.e.
  ``:filtered_attribute:memberOf`` will mean that only the values of memberOf
  which match ``group_filter`` will be considered. An example value
  of the ``memberOf`` attribute would be: ``CN=mygroup,OU=My Group,OU=LabUsers,
  DC=openmicroscopy,DC=org``
- If prefixed with ``:dn_attribute:``, then the rest of the string is taken to
  be an attribute all of whose values will be taken as group distinguished
  names. For example, ``omero.ldap.new_user_group=:dn_attribute:memberOf``
  would add a user to all the groups named by memberOf, where the name of the
  group is mapped via ``group_mapping``. You can prefix this value with
  ``filtered_`` to have the ``group_filter`` applied to the attribute values,
  i.e. ``:filtered_dn_attribute:memberOf`` will mean that only the values of
  memberOf which match ``group_filter`` will be considered. An example value
  of the ``memberOf`` attribute would be: ``CN=mygroup,OU=My Group,OU=LabUsers,
  DC=openmicroscopy,DC=org``

  Note that if an
  attribute specified in :property:`omero.ldap.group_mapping` does not constitute a
  part of the Distinguished Name (DN) as determined by your LDAP server then it
  can only be found by using ``:attribute:`` or ``:filtered_attribute:``
  instead. Typical attributes that comprise the DN are: DC, CN, OU, O, STREET,
  L, ST, C and UID.
- If prefixed with ``:query:``, then the rest of the value is taken as a query
  to be AND'ed to the group filter. In the query, values from the user such as
  "@{cn}", "@{email}", or "@{dn}" can be used as place holders.
- If prefixed with ``:bean:``, then the rest of the string is the name of a
  Spring bean which implements the NewUserGroupBean interface. See the
  developer documentation :doc:`/developers/Server/Ldap` for more info.

Compound Filters
^^^^^^^^^^^^^^^^

.. note:: OMERO uses standard
    `RFC 2254 LDAP filters <http://www.faqs.org/rfcs/rfc2254.html>`_, so they
    must conform to that syntax and are only able to do what those filters can
    do. You can test the filters via ldapsearch on your OMERO server (assuming
    you have the OpenLDAP binaries installed).
    
    If you are using OpenLDAP make sure your directory has the ``memberOf``
    attribute correctly configured. Some versions of ApacheDS do not support
    ``memberOf`` at all.

Both the ``user_filter`` and the ``group_filter`` can contain any valid LDAP
filter string. These must be a valid filter in themselves. e.g.

::

   omero.ldap.user_filter=(|(ou=Queensland Brain Institute)(ou=Ageing Dementia Research))

The "|" operator (read: "OR") above allows members of two organizational units
to login to OMERO. Expanding the list allows concentric "rings" of more and
more OU's granular access to OMERO.

::

   omero.ldap.group_filter=(&(objectClass=groupOfNames)(mail=omero.flag))

The "&" operator (read: "AND") produces a filter that will only match groups that have
the ``mail`` attribute set to the value ``omero.flag``. When combined with
the ``group_mapping``, the final query would be
``(&(&(objectClass=groupOfNames)(mail=omero.flag))(cn=[group name]))``

This is the same as the query
``(&(objectClass=groupOfNames)(mail=omero.flag)(cn=[group name]))`` but setting
``group_filter`` to ``(objectClass=groupOfNames)(mail=omero.flag)`` is not valid
as that is not a valid filter on its own.

To restrict the list of groups to just the ones returned by the above query, the
following setting is also required to remove unmatched groups:

::

   omero.ldap.new_user_group=:filtered_dn_attribute:memberOf

.. _case-sensitivity:

Case sensitivity
----------------

By default, the LDAP plugin is case-sensitive i.e. it will treat the usernames
JSmith and jsmith as two different users. You can remove case sensitivity
using::

   omero config set omero.security.ignore_case true

.. warning:: Enabling this option will affect **all**, even non-LDAP,
   usernames in your OMERO system. It is the system administrator's
   responsibility to handle any username clashes which may result.
   Making non-LDAP usernames lowercase is required. Non-LDAP users with
   uppercase characters in their username will not be able to log in and
   will not appear in some administrative tools.

   ``UPDATE experimenter SET omename = lower(omename);`` can be used on
   your database to make this change to all users if desired. This operation
   is irreversible.

LDAP over |SSL|
---------------

If you are connecting to your server over |SSL|,
that is, if your URL is of the form ``ldaps://ldap.example.com:636`` you may
need to configure a key and trust store for Java. See the
:doc:`server-security` page for more information.

.. _synchronizing-ldap:

Synchronizing LDAP on user login
--------------------------------

This feature allows for LDAP to be considered the authority on user/group
membership. With the following setting enabled, each time a user logs in to
OMERO their LDAP groups will be read from the LDAP server and reflected in
OMERO::

    omero config set omero.ldap.sync_on_login true

Admin actions carried out in the clients may not survive this
synchronization e.g. if an admin has removed an LDAP user from an LDAP group
in the UI, the user will be re-added to the group when logging in again after
the synchronization.

.. note:: This applies to groups created by LDAP in OMERO 5.1.x. Groups
    created in older versions of OMERO will not be registered as LDAP groups
    if you have manually altered their membership, even if the membership now
    matches the LDAP group.
    
    :program:`omero ldap setdn true --group-name $NAME` can be used to make these
    previous OMERO groups into LDAP groups.

.. _legacy_password_providers:

Legacy password providers
-------------------------

The primary component of the LDAP plugin is the LdapPasswordProvider, which is
responsible for creating users, checking their passwords, and adding them to
or removing them from groups. The default password provider is the
``chainedPasswordProvider`` which first checks LDAP if LDAP is enabled, and
then checks JDBC. This can explicitly be enabled by executing the system admin
command:

::

    omero config set omero.security.password_provider chainedPasswordProvider

When the LDAP password provider implementation changes, previous versions can
be configured as necessary.

- ``chainedPasswordProviderNoSalt``

  The ``chainedPasswordProviderNoSalt`` uses the version of the JDBC
  password provider without password salting support as available in the
  OMERO 4.4.x series. To enable it, use:

  ::

    omero config set omero.security.password_provider chainedPasswordProviderNoSalt

- ``chainedPasswordProvider431``

  With the 431 password provider, the user filter is only checked on first
  login and not kept on subsequent logins. This allows for an OMERO admin
  to change the username of a user in omero to be different than the one
  kept in LDAP. To enable it, use:

  ::

    omero config set omero.security.password_provider chainedPasswordProvider431

.. seealso::

    :doc:`unix/server-installation`
        Installation guide for OMERO.server under UNIX-based platforms

    :doc:`server-security`
        Security pages for OMERO.server

    :doc:`/developers/Server/Ldap`
        Developer documentation on extending the LDAP plugin yourself.

    :forum:`What are your LDAP requirements? <viewtopic.php?f=5&t=14>`
        Forum discussion if you have LDAP requirements that are not covered by the above configuration

    JNDI referrals documentation
        https://docs.oracle.com/javase/jndi/tutorial/ldap/referral/jndi.html

Active Directory
----------------
`Active Directory <https://en.wikipedia.org/wiki/Active_Directory>`_ (AD) supports
a form of LDAP and can be used by OMERO like most other directory services.

In AD, the `Domain Services <https://msdn.microsoft.com/en-us/library/aa362244(v=vs.85).aspx>`_ (DS)
'forest' is a complete instance of an Active Directory which contains one or more domains. Querying
a particular Domain Service will yield results which are local to that domain only. In an environment
with just one domain it is possible to use the default configuration instructions for OMERO LDAP. If
there are multiple domains in the forest then it is necessary to query the
:ref:`ad_global_catalogue` to enable querying across all of them.

.. _ad_global_catalogue:

Global Catalogue
^^^^^^^^^^^^^^^^
In an AD DS forest, a `Global Catalogue <https://technet.microsoft.com/en-us/library/cc728188(v=ws.10).aspx>`_
provides a central repository of all the domain information from all of the domains. This can be queried in
the same way as a specific Domain Service using LDAP, but it runs on different ports; 3268 and 3269 (SSL).

-  LDAP AD Global Catalogue server URL string

   ::

       omero config set omero.ldap.urls ldap://ldap.example.com:3268

   .. note::

       A |SSL| URL above should look like this:
       ldaps://ldap.example.com:3269
