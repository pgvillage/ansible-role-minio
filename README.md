Walg
=========

Minio is a tool to create a S3 bucket out of a filesystem.
Minio can do a whole lot more, but this is what Minio is to PgVillage.
This role installs and configures minio.
This role is part of PgVillage, which is an opinated PostgreSQL deployment for Virtual Machines.

Requirements
------------

This role aims at using an RPM from the MannemSolutions repo.

Role Variables
--------------

Please see [defaults](https://github.com/pgvillage/ansible-role-minio/blob/main/defaults/main.yml) for all variables


Dependencies
------------

No dependencies


Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - pgvillage.minio

License
-------

PostgreSQL

Author Information
------------------

PgVillage is an Open Community.
Main contributor is Nibble-IT.
