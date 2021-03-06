# hybrid SQL and LDAP backend for OpenStack Keystone #

This is a proof of concept to allow LDAP authentication while using the SQL backend for all the usual operations. No users or groups are copied from LDAP. LDAP users are assigned a default role and tenant when they first login if they don't already have one (user_project_metadata table). For granting roles to users (`keystone user-role-add`), only the user id from LDAP is inserted into the SQL backend.

The code in this branch has only been tested on the stable/grizzly branch of openstack keystone!

* * *

## Installation ##

Since this backend relies on both the LDAP and SQL backends, you have to configure both beforehand. Use the usual configuration options found in /etc/keystone/keystone.conf. However, from the LDAP backend's config, only the ldap.user* options will be used by the hybrid backend (so no tenant/role options).

You should try to see that user authentication works fine with the LDAP backend before trying on the hybrid backend. Also make sure that `keystone user-list` works using the LDAP identity backend.

Remove the database Foreign Key constraint which disallows user_id values which are not present in the SQL backend:

```SQL
$ pgsql keystone -U keystone -W
ALTER TABLE user_project_metadata DROP CONSTRAINT user_project_metadata_user_id_fkey;
```

or for MySQL:

```SQL
$ mysql keystone
SHOW CREATE TABLE user_project_metadata;
-- look for the constraint name referencing user.id
ALTER TABLE user_project_metadata DROP FOREIGN KEY user_project_metadata_ibfk_1;
```

Set the identity backend to `hybrid` (it will use both the LDAP and the SQL backends under the hood):

```
[identity]
driver = keystone.identity.backends.hybrid.Identity
```

Copy the `hybrid.py` file to the `keystone/identity/backends/` folder of your installation (e.g. `/usr/lib/python/site-packages/keystone/identity/backends/hybrid.py`). Edit the file and set the `DEFAULT_TENANT`, `DEFAULT_ROLE` and `DEFAULT_DOMAIN` constants at the top of the file. Make sure that the tenant and role set here also exist in keystone or create them otherwise!

Restart keystone.

Now you can assign custom roles to users in LDAP. Make user you use the LDAP user-id returned by the `keystone user-list` query.

```
keystone user-role-add --user-id=12345 --role-id <role-id> --tenant-id <tenant-id>
```
