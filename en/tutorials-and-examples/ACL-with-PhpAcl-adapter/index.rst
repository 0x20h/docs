An example application using the PhpAcl adapter
###############################################

In this tutorial you will create a simple e-learning application that shows some use cases for 
the ``PhpAcl`` adapter. ``PhpAcl`` uses a static PHP array to define the ARO groupings, and the ACO
paths.  This is an ideal implementation when you have a permission system that
doesn't need to be changed on the fly. Its also faster than DbAcl.

Introduction on Access Control Lists
====================================

In both ``PhpAcl``, and ``DbAcl`` ARO and ACO nodes are stored in tree
structures.  This allows for permissions to cascade down the tree. For the
following example, we'll assume an simple e-learning application with a few user
groups.  We'll also assume that we're using our ACL with
:php:class:`AuthComponent` with :ref:`ActionsAuthorize <authorization-objects>`.  Our ACO tree, will be
comprised of our controllers and their actions:

* controllers

  * Lessons

    * index
    * add
    * edit
    * view
    * delete

  * Courses

    * index
    * add
    * edit
    * delete

  * Students

    * index
    * add
    * edit
    * delete

While our ARO tree would look like:

* users

  * Administrators

    * Bob
    * Betty

  * Teachers

    * Fred
    * Felicity

  * Students

    * Joe
    * Jessica


With these tree structures, we can define permissions that apply to groups of users, or
just a single one.  This reduces duplication in your permissions, allows
permissions to cascade for any given path, and gives you a powerful granular
permission system.  When permissions are checked, each path is traversed until
an allow/deny rule is reached.  For example, we'll grant a few permissions, and
then check how permissions are resolved:

* controllers **Deny: users, Grant: Administrators**

  * Lessons **Grant: Teachers**

    * index **Grant: Students**
    * add
    * edit
    * view **Grant Students**
    * delete 

  * Courses **Grant: Teachers**

    * index **Grant: Students**
    * add
    * edit
    * delete

  * Students

    * index **Grant: Teachers**
    * add **Grant: users**
    * edit **Grant Students**
    * delete

When checking if ``User/Joe`` can access ``controllers/Courses/add``
the following happens:

* The tree for each path is generated, and the terminal nodes are fetched.
* A permission lookup is done for the ``Joe`` and ``add`` nodes.
* Since the previous lookup failed, permissions lookups are done for each parent
  node in the tree.
* Since the only permission set is the deny rule for ``users`` and
  ``controllers`` the acl check fails.

Setting up Models
=================

::

  CREATE TABLE users (
    id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    password CHAR(40) NOT NULL,
    group_id INT(11) NOT NULL,
    created DATETIME,
    modified DATETIME
  );


  CREATE TABLE groups (
    id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created DATETIME,
    modified DATETIME
  );


  CREATE TABLE courses (
    id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    created DATETIME,
    modified DATETIME
  );

  CREATE TABLE lessons (
    id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    course_id INT(11) NOT NULL,
    name VARCHAR(100) NOT NULL,
    body TEXT,
  );


Use the :doc:`cake bake </console-and-shells/code-generation-with-bake>` commands to setup models, controllers and views for the application. For
simplicity you can use scaffolding.

Setting up Authentication
=========================

To setup Authentication, please follow the instructions from the :ref:`Simple ACL controlled Application <preparing-authentication>` tutorial.

.. _configuring-phpacl:

Setting up the PhpACL Adapter
=============================

At this point you have added the database tables and generated controllers, models and views. 
You can save users and passwords and users need to authenticate themselves using the ``/users/login`` action.
Now we need to restrict access to certain actions, allowing those only to a subset of registered users.
To enable the :php:class:`PhpAcl` adapter set the ``Acl.classname`` property in 
``app/Config/core.php`` ::

	<?php
	//...
	//Configure::write('Acl.classname', 'DbAcl');
	//Configure::write('Acl.database', 'default');
	Configure::write('Acl.classname', 'PhpAcl');

Setting up permissions
----------------------

Let's setup ``app/Config/acl.php`` to reflect the access rules of our e-learning 
application. We assume that the user data is stored in a ``username`` and a ``group_id``
column of a ``User`` model. In order to map a ``User`` record to a role defined in :php:class:`PhpAcl` we need to 
tell the adapter how the obtain the relevant information (the default map is
``User => User/username`` and ``Role => User/role``)::

    <?php
    $config['map'] = array(
        'User' => 'User/username',
        'Role' => 'User/group_id',
    );

If a ``User`` array with ``username`` and ``group_id`` fields is passed as ARO
(e.g. ``array('User' => array('username' => 'Fred', 'group_id' => 2)``) :php:class:`PhpAcl` internally
will lookup if a role ``User`` is defined for the provided ``username``. If no role matches, 
:php:class:`PhpAcl` will check if a role ``Role`` is defined for the provided ``group_id``. If no role can be found, the ARO will
be resolved to the default role ``Role/default``. Because the roles are given as model IDs we can (optionally) 
define some aliases for our ``group_ids`` to make
definition of roles and rules easier to read::

    <?php
    $config['alias'] = array(
        'Role/1' => 'Role/Administrator',   // group_id 1 == Administrator
        'Role/2' => 'Role/Teacher',         //          2 == Teacher
        'Role/3' => 'Role/Student',         //          3 == Student
    );

Now we can setup the roles. Roles are defined as keys, inherited roles as values. Inherited roles can be defined as a
comma separated list or as array, ``null`` values indicate root nodes::

    <?php
    // AROs
    $config['roles'] = array(
        'Role/Administrator' => null,
        'Role/Teacher' => 'Role/default',
        'Role/Student' => array('Role/default'),
    );

Now let's setup rules. The rules array can contain two keys, ``allow`` and ``deny``. For our simple 
example we'll only need to define ``allow`` rules as by default every access controlled 
object is denied:: 
    
    <?php
    // ACOs
    $config['rules']['allow'] = array(
        '/*' => 'Role/Administrator',
        '/controllers/Lessons' => 'Role/Teacher',
        '/controllers/Lessons/(index|view)' => 'Role/Student',
        '/controllers/Courses' => 'Role/Teacher',
        '/controllers/Courses/index' => 'Role/Student',
        '/controllers/Students/index' => 'Role/Teacher',
        '/controllers/Students/add' => 'Role/default',
        '/controllers/Students/edit' => 'Role/Student',
    );

Using ACO Patterns
------------------

As you can see from the example above, ACOs (array keys of rules) can be defined by using wildcards.
PhpAcl splits ACOs by ``/`` and then treats every token as a regular expression after replacing
``*`` with ``.*``. When checking access, the requested ACO is split analogous and each token is
matched against its respective rule token. Example::

    <?php
    // in some action
    public function index() {
        $this->Acl->Aro->addRole(array('User/Felicity' => 'Role/Teacher, Role/default'));
        $this->Acl->Aro->addRole(array('User/Fred' => array('Role/Teacher', 'Role/default')));

        $this->Acl->allow('/controllers/*/manager_[a-zA-Z]+', 'Role/Teacher');
        $this->Acl->deny('/controllers/Courses/manager_delete', 'Role/Teacher');
        $this->Acl->deny('/controllers/Courses/manager_confirm', 'User/Felicity');

        $this->Acl->check('Felicity', '/controllers/Foo/manager_bar'); // true
        $this->Acl->check('Felicity', '/controllers/Courses/manager_delete'); // false
        $this->Acl->check('Felicity', '/controllers/Courses/manager_confirm'); // false
        $this->Acl->check('Fred', '/controllers/Courses/manager_confirm'); // true
    }

The ``allow()`` call grants every ``Teacher`` access to all actions starting with ``manager_`` for every 
controller. The ``deny()`` calls repeal the grants for the ``manager_delete`` 
action in the ``Courses`` controller. Additionally ``Felicity`` would not be allowed to 
access the ``manager_confirm`` action.

Runtime options
---------------

Additional options can be passed to the :php:class:`PhpAcl` instance::

    <?php
        // in AppController
        public $components = array(
            // ...
            'Acl' => array(
                'adapter' => array(
                    'config' => '/my/acl.php',
                    'policy' => PhpAcl::ALLOW,
                ),
            ),
        );

The ``config`` key refers to the ACL definition file and will be passed to :php:class:`PhpReader`. 
Setting ``policy`` to ``PhpAcl::ALLOW`` follows a blacklist approach where you would only specify
``deny`` rules, while by default every ACO is allowed. 



.. meta::
    :title lang=en: An example application using the PhpAcl adapter
    :keywords lang=en: access control list,example application,request objects,request object,acls,lingo,web service,computer system,cakephp,control objects,control object
