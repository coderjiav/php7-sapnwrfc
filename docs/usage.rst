Usage
=====

This guide describes commonly used functions of the ``php7-sapnwrfc`` extension.
For a more detailed overview of available methods, refer to the :doc:`API overview </api>`. 

For getting started right away, refer to the :ref:`quick-start`.

Connecting to a SAP system
--------------------------

To connect to a SAP system, we create a new instance of ``SAPNWRFC\Connection``,
passing connection parameters:

.. code-block:: php

    <?php
    $parameters = [
        'ashost' => 'my.sap.system.local',
        'sysnr'  => '00',
        'client' => '123',
        'user' => 'DEMO',
        'passwd' => 'XXXX',
    ];

    // connect
    $connection = new SAPNWRFC\Connection($parameters);

    // do work here

    // close the connection
    $connection->close();

If the connection attempt fails, a ``SAPNWRFC\ConnectionException`` is thrown
with details about the error.

We can also use a ``sapnwrfc.ini`` file to specify connection details. See
``SAPNWRFC\Connection::setIniPath($path)`` and ``SAPNWRFC\Connection::reloadIniFile()``
in the :doc:`API overview </api>` for details.

For available connection parameters refer to the `sapnwrfc.ini parameter overview <https://help.sap.com/saphelp_nw73ehp1/helpdata/en/48/a88c805134307de10000000a42189b/frameset.htm?frameset=/en/48/a994a77e28674be10000000a421937/frameset.htm>`_.

After we are done using the connection, it is recommended to close the connection
using ``SAPNWRFC\Connection::close()``.

Calling remote function modules
-------------------------------

Looking up the function module
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Before we can call a remote function module, we have to lookup the desired
function module to retrieve a ``SAPNWRFC\RemoteFunction`` object. This is accomplished
with the ``SAPNWRFC\Connection::getFunction($functionName)`` method. We simply pass
the name of the remote function module:

.. code-block:: php

    <?php
    
    $remoteFunction = $connection->getFunction('RFC_PING');

If the lookup is successful, an object of type ``SAPNWRFC\RemoteFunction`` is returned
which can be used for invoking the function.

If the function lookup fails, a ``SAPNWRFC\FunctionCallException`` is thrown.

Calling the function module
^^^^^^^^^^^^^^^^^^^^^^^^^^^

After we have retrieved a ``SAPNWRFC\RemoteFunction`` object, we can invoke
the function module by using its ``invoke()`` method.

To invoke the function module ``RFC_PING``, we can simply call ``invoke()``
without parameters.

.. code-block:: php

    <?php

    $returnValue = $remoteFunction->invoke();

If the function module returns parameters, ``invoke()`` returns them. On any
errors, a ``SAPNWRFC\FunctionCallException`` is thrown.

Parameters and return values
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In practice, most of the remote function modules will require us to pass
parameters and/or will return parameters after invocation.

Passing parameters is possible using the ``$parameters`` array of the remote
function's ``invoke()`` method. Parameters are passed as arrays, with the
key being the name of the parameter and the value the value to pass.

The same is true for return values. If a remote function module returns parameters,
they will be available in the array returned from ``invoke()``.

.. note::

    ABAP function modules know four different types of parameters:

    - ``IMPORT``: set by the client.
    - ``EXPORT``: set by the server.
    - ``CHANGING``: set by the client and can be modified by the server.
    - ``TABLE``: set by the client and can be modified by the server.

To call a function module ``STFC_CHANGING`` that requires input parameters as well
as returns parameters to the caller, we can do as follows:

.. code-block:: php

    <?php

    $function = $connection->getFunction('STFC_CHANGING');
    $result = $function->invoke([
        'START_VALUE' => 0,
        'COUNTER' => 1,
    ]);

    /*
    $result looks like this:

    array(2) {
      ["COUNTER"] => int(2)
      ["RESULT"] => int(1)
    }
    */

The types of the parameters and return values are mapped to standard PHP types.

Parameter type mappings
^^^^^^^^^^^^^^^^^^^^^^^

Remote function modules execute ABAP code and therefor use ABAB data types. This extension
maps between RFC data types and builtin PHP data types as follows:

+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| ABAP type | RFC type | PHP type  | Meaning                                           | Notes                                          |
+===========+==========+===========+===================================================+================================================+
| C         | CHAR     | string    | Text field (alphanumeric characters)              | right-padded with blanks; see ``rtrim`` option |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| D         | DATE     | string    | Date field (format: YYYYMMDD)                     |                                                |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| T         | TIME     | string    | Time field (format: HHMMSS)                       |                                                |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| X         | BYTE     | string    | Hexadecimal field                                 | use ``hex2bin()`` to convert to binary         |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| N         | NUM      | string    | Numeric text field                                |                                                |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| STRING    | STRING   | string    | String (dynamic length)                           |                                                |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| XSTRING   | BYTE     | string    | Hexadecimal string (dynamic length)               | use ``hex2bin()`` to convert to binary         |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| I         | INT      | integer   | Integer                                           | INT1 and INT2 are also mapped to integer       |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| P         | BCD      | double    | Packed number / BCD                               |                                                |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+
| F         | FLOAT    | double    | Floating point number                             |                                                |
+-----------+----------+-----------+---------------------------------------------------+------------------------------------------------+

Additionally, there are also tables and structures:

- A structure is mapped to an associative array, with the keys being the field names
  and the values the field values.
- A table is an array of structures.

Passing options when calling function modules
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When calling function modules using ``RemoteFunction::invoke()`` a second parameter can
be passed specifying options for the function call.

.. code-block:: php

    <?php

    // ...
    $options = [
        'rtrim' => true
    ];

    $function->invoke($parameters, $options);

The following options are available:

rtrim
    In ABAP, there are two ways to store strings: as fixed length string type C or as dynamic
    length type STRING. When using type C strings are right-padded with blanks, if the string
    is shorter than the predefined length. To unify the extensions behaviour with strings, the
    ``rtrim`` option is available. If set to ``true``, type C strings are right-trimmed before
    being returned.

    *Default:* ``false``

Activating/Deactivating parameters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The *SAP NW RFC library* supports the activation and deactivation of parameters
of a function module. This is especially useful if a function module has a lot of
(possibly big) return values that you are not interested in.

To activate or deactivate a parameter, we call the method 
``SAPNWRFC\RemoteFunction::setParameterActive($parameterName, $isActive)``
on the remote function object. We can use ``SAPNWRFC\RemoteFunction::isParameterActive($parameterName)``
to check if a parameter is active or not.

.. code-block:: php

    <?php

    $function = $connection->getFunction('STFC_STRUCTURE');

    $function->setParameterActive('IMPORTSTRUCT', false);
    $function->setParameterActive('RFCTABLE', false);

    $function->isParameterActive('IMPORTSTRUCT'); // returns false

    // we don't need to pass parameters when invoking the function module,
    // as we deactivated all of them
    $result = $function->invoke([]);

    // $result will not contain the 'RFCTABLE' parameter

    $function->setParameterActive('RFCTABLE', true);
    $function->isParameterActive('IMPORTSTRUCT'); // returns true

    // we need to pass the 'RFC_TABLE' parameter now
    $result = $function->invoke([
        'RFCTABLE' => [],
    ]);

Function module details
^^^^^^^^^^^^^^^^^^^^^^^

The ``SAPNWRFC\RemoteFunction`` object defines a ``name`` property containing
the name of the function module it represents.

Additionally, a property for each parameter of the function module is defined
on the object that can be used to get detailed information about the parameter.

Trace levels
------------

The *SAP NW RFC library* allows the creation of trace files to troubleshoot
connection and/or function call problems.

We can set the desired trace level via the ``trace`` parameter when
establishing the connection or change it at any time using the
``SAPNWRFC\Connection::setTraceLevel($level)`` method.

The extension defines constants on the ``SAPNWRFC\Connection`` class 
for the four trace levels (from least to most verbose): ``TRACE_LEVEL_OFF``,
``TRACE_LEVEL_BRIEF``, ``TRACE_LEVEL_VERBOSE``, ``TRACE_LEVEL_FULL``.

Additionally, we can set the directory for the generated trace file
using ``SAPNWRFC\Connection::setTraceDir($path)``. The default location
for the trace file is the current working directory.

Getting the version
-------------------

The extension provides the methods ``SAPNWRFC\Connection::version()`` and
``SAPNWRFC\Connection::rfcVersion()`` for getting the extension version
and the RFC SDK version in use.

Both methods return a string formatted as ``MAJOR.MINOR.PATCH`` (for example 1.1.3);

Exceptions
----------

If an error occurs during any interaction, an exception is thrown with error details.
Currently, there are two exception classes:

- ``SAPNWRFC\ConnectionException`` for any errors that concern the connection itself.
- ``SAPNWRFC\FunctionCallException`` for errors that stem from function module invocations.

Both exception classes extend the base exception class ``SAPNWRFC\Exception``, which
adds an additional ``getErrorInfo()`` method to the standard methods and properties
provided by ``RuntimeException``.

``getErrorInfo()`` returns an array with detailed error information and contains
at least the keys ``code``, ``key`` and ``message``. 

Depending on the type of error, the following additional keys might be available
with detailed information: ``abapMsgClass``, ``abapMsgType``, ``abapMsgNumber``, 
``abapMsgV1``, ``abapMsgV2``, ``abapMsgV3``, ``abapMsgV4``.
