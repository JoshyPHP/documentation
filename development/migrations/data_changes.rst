============
Data changes
============

Data changes are done in Migrations by creating an array that will be parsed by
``\phpbb\db\migrator::run_step()``

Basics
======

When performing data changes, your Migrations class should contain at least one
function, update_data(). You may optionally provide a function revert_data(),
which will be run when the migration is removed.

update_data
===========

``update_data()`` will be called as the **second** step, after
``update_schema()`` when installing a migration. This makes any database data
changes you need when installing the migration.

.. code-block:: php

    public function update_data()
    {
        return array();
    }

revert_data
===========

``revert_data()`` will be called when a Migration must be uninstalled (this
could happen if the user wants/needs to revert changes by the Migration, a
dependency is reverted, or the installation fails and the entire Migration is
removed).

This function is entirely optional, **most data is automatically reverted.** All
calls to the Migration Tools are automatically reverted. The only thing this
should do is handle reverting any custom functions that were run if it is
absolutely needed.

.. code-block:: php

    public function revert_data()
    {
        return array();
    }


What to return
==============

An array of arrays containing instructions.

These instructions can include calls to Tools. For any config settings,
permission related commands, and modules, please see the related documentation
under :doc:`tools/index`.


If (Conditional)
================

If call allows you to create a basic if statement which will be checked and,
if true, the attached statement will be parsed.

How it works
------------

.. code-block:: php

    array('if', array(
        true, // Some statement that is either true or false
        array(/* Call to make if the statement is true */),
    )),

Examples
--------

if config "captcha_gd" is true, update "captcha_plugin" with "phpbb_captcha_gd"

.. code-block:: php

    array('if', array(
        ($this->config['captcha_gd']),
        array('config.update', array('captcha_plugin', 'phpbb_captcha_gd')),
    )),

if config "allow_avatar_upload" or "allow_avatar_local" is true, update
"allow_avatar" with value "1"


.. code-block:: php

    array('if', array(
        ($this->config['allow_avatar_upload'] || $this->config['allow_avatar_local']),
        array('config.update', array('allow_avatar', 1)),
    )),

Custom
======

Custom calls allow you to specify the callable to your own function to be called.

How it works
------------

.. code-block:: php

    array('custom', array(
        array(/* Callable function, sent to call_user_func_array */)
    )),

Example
-------

Call a function within the migrations file named some_function

.. code-block:: php

    array('custom', array(
        array(&$this, 'some_function')
    )),

**Note:** the function called, must be public accessible

Multi step processes
--------------------

If you have a function that needs to be called multiple times to complete,
returning anything except null or true will cause the function to be called
until null or true is returned.

**Note:** This should be used when something needs to be run that can take
longer than the time limit (for example, resyncing topics).

Example
-------

.. code-block:: php

    public function update_data()
    {
        return array(
            array('custom', array(
                array(&$this, 'some_function')
            )),
        );
    }

    // $value is equal to the value returned on the previous call (false if this is the first time it is run)
    public function some_function($value)
    {
        $limit = 500;
        $i = 0;

        // Select all topics, starting at $value, limit $limit
        while ($topic = fetchrow)
        {
            $i++;

            // Do something
        }

        if ($i < $limit)
        {
            // There are no more topics, we are done
            return;
        }

        // There are still more topics to query, return the next start value
        return $value + $limit;
    }

Examples
========

From ``\phpbb\db\migration\data\v310\dev``

.. code-block:: php

    public function update_data()
    {
        return array(
            array('config.update', array('search_type', 'phpbb_search_' . $this->config['search_type'])),

            array('config.add', array('fulltext_postgres_ts_name', 'simple')),
            array('config.add', array('fulltext_postgres_min_word_len', 4)),
            ...

            array('permission.add', array('u_chgprofileinfo', true, 'u_sig')),

            array('module.add', array(
                'acp',
                'ACP_GROUPS',
                array(
                    'module_basename'    => 'acp_groups',
                    'modes'                => array('position'),
                ),
            )),
            ...

            // Module will be renamed later
            array('module.add', array(
                'acp',
                'ACP_CAT_STYLES',
                'ACP_LANGUAGE'
            )),

            array('module.remove', array(
                'acp',
                false,
                'ACP_TEMPLATES',
            )),

            array('custom', array(array($this, 'rename_module_basenames'))),
            array('custom', array(array($this, 'rename_styles_module'))),
            ...

            array('config.update', array('version', '3.1.0-dev')),
        );
    }

    public function rename_styles_module()
    {
        // Rename styles module to Customise
        $sql = 'UPDATE ' . MODULES_TABLE . "
            SET module_langname = 'ACP_CAT_CUSTOMISE'
            WHERE module_langname = 'ACP_CAT_STYLES'";
        $this->sql_query($sql);
    }

    public function rename_module_basenames()
    {
        // rename all module basenames to full classname
        $sql = 'SELECT module_id, module_basename, module_class
            FROM ' . MODULES_TABLE;
        $result = $this->db->sql_query($sql);

        while ($row = $this->db->sql_fetchrow($result))
        {
            $module_id = (int) $row['module_id'];
            unset($row['module_id']);

            if (!empty($row['module_basename']) && !empty($row['module_class']))
            {
                // all the class names start with class name or with phpbb_ for auto loading
                if (strpos($row['module_basename'], $row['module_class'] . '_') !== 0 &&
                    strpos($row['module_basename'], 'phpbb_') !== 0)
                {
                    $row['module_basename'] = $row['module_class'] . '_' . $row['module_basename'];

                    $sql_update = $this->db->sql_build_array('UPDATE', $row);

                    $sql = 'UPDATE ' . MODULES_TABLE . '
                        SET ' . $sql_update . '
                        WHERE module_id = ' . $module_id;
                    $this->sql_query($sql);
                }
            }
        }

        $this->db->sql_freeresult($result);
    }
