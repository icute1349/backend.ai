JSON Object References
======================

.. _paging-query-object:

Paging Query Object
-------------------

It describes how many items to fetch for object listing APIs.
If ``index`` exceeds the number of pages calculated by the server, an empty list is returned.

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description
   * - ``size``
     - ``int``
     - The number of items per page.
       If set zero or this object is entirely omitted, all items are returned and ``index`` is ignored.
   * - ``index``
     - ``int``
     - The page number to show, zero-based.

.. _paging-info-object:

Paging Info Object
------------------

It contains the paging information based on the paging query object in the request.

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description
   * - ``pages``
     - ``int``
     - The number of total pages.
   * - ``count``
     - ``int``
     - The number of all items.

.. _keypair-item-object:

KeyPair Item Object
-------------------

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description
   * - ``accessKey``
     - ``slug``
     - The access key part.
   * - ``isActive``
     - ``bool``
     - Indicates if the keypair is active or not.
   * - ``totalQueries``
     - ``int``
     - The number of queries done via this keypair. It may have a stale value.
   * - ``created``
     - ``datetime``
     - The timestamp when the keypair was created.

.. _keypair-props-object:

KeyPair Properties Object
-------------------------

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description
   * - ``isActive``
     - ``bool``
     - Indicates if the keypair is activated or not.
       If not activated, all authentication using the keypair returns 401 Unauthorized.
       When changed from ``true`` to ``false``, existing running kernel sessions continue to run but any requests to create new kernel sessions are refused.
       (default: ``true``)
   * - ``concurrecy``
     - ``int``
     - The maximum number of concurrent kernel sessions allowed for this keypair.
       (default: ``5``)
   * - ``ML.clusterSize``
     - ``int``
     - Sets the number of instances clustered together when launching new machine learning kernel sessions. (default: ``1``)
   * - ``ML.instanceMemory``
     - ``int`` (MiB)
     - Sets the memory limit of each instance in the cluster launched for new machine learning kernel sessions. (default: ``8``)

The enterprise edition offers the following additional properties:

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description
   * - ``cost.automatic``
     - ``bool``
     - If set ``true``, enables automatic cost optimization (BETA).
       With supported kernel types, it automatically suspends or resize the kernel sessions not to exceed the configured cost limit per day.
       (default: ``false``)
   * - ``cost.dailyLimit``
     - ``str``
     - The string representation of money amount as decimals.
       The currency is fixed to USD. (default: ``"50.00"``)

.. _batch-execution-query-object:

Batch Execution Query Object
----------------------------

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description
   * - ``build``
     - ``str``

     - The bash command to build the main program from the given uploaded files.

       If this field is not present, an empty string or ``null``, it skips the build step.

       If this field is a constant string ``"*"``, it will use a default build script provided
       by the kernel.
       For example, the C kernel's default Makefile adds all C source files
       under the working directory and copmiles them into ``./main``
       executable, with commonly used C/link flags: ``"-pthread -lm -lrt -ldl"``.

   * - ``exec``
     - ``str``

     - The bash command to execute the main program.

       If this is not present, an empty string, or ``null``, the server only
       performs the build step and ``options.buildLog`` is assumed to be
       ``true`` (the given value is ignored).

.. note::

   A client can distinguish whether the current output is from the build phase
   or the execution phase by whether it has received ``build-finished`` status
   or not.

.. note::

   All shell commands are by default executed under ``/home/work``.
   The common environment is:

   .. code-block:: text

      TERM=xterm
      LANG=C.UTF-8
      SHELL=/bin/bash
      USER=work
      HOME=/home/work

   but individual kernels may have additional environment settings.

.. warning::

   The shell does NOT have access to sudo or the root privilege.
   Though, some kernels may allow installation of language-specific packages in
   the user directory.

   Also, your build script and the main program is executed inside
   Sorna Jail, meaning that some system calls are blocked by our policy.
   Since ``ptrace`` syscall is blocked, you cannot use native debuggers
   such as gdb.

   This limitation, however, is subject to change in the future.

Example:

.. code-block:: json

   {
     "build": "gcc -Wall main.c -o main -lrt -lz",
     "exec": "./main"
   }


.. _execution-result-object:

Execution Result Object
-----------------------

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description

   * - ``runId``
     - ``str``
     - The user-provided run identifier.
       If the user has NOT provided it, this will be set by the API server upon the first execute API call.
       In that case, the client should use it for the subsequent execute API calls during the same run.

   * - ``status``
     - ``enum[str]``

     - One of ``"continued"``, ``"waiting-input"``, ``"finished"``, or ``"build-finished"``.
       See more details at :ref:`code-execution-model`.

   * - ``exitCode``
     - ``int | null``
     - The exit code of the last process.
       This field has a valid value only when the ``status`` is ``"finished"`` or ``"build-finished"``.
       Otherwise it is set to ``null``.

       For batch-mode kernels and query-mode kernels *without* global context support,
       ``exitCode`` is the return code of the last executed child process in the kernel.
       In the execution step of a batch mode run, this is always 127 (a UNIX shell common practice for "command not found")
       when the build step has failed.

       For query-mode kernels with global context support, this value is always zero,
       regardless of whether the user code has caused an exception or not.

       A negative value (which cannot happen with normal process termination) indicates a Backend.AI-side error.

   * - ``console``
     - .. code-block:: text

          list[
            tuple[
              enum[str], *
            ]
          ]

     - Contains a list of console output items.
       Each item is a pair of the item type (``enum[str]``) and its value (``*``).
       See more details at :ref:`handling-console-output`.

   * - ``options``
     - ``object``

     - An object containing extra display options.  If there is no options indicated by the kernel, this field is ``null``.
       When ``result.status`` is ``"waiting-input"``, it has a boolean field ``is_password`` so that you could use
       different types of text boxes for user inputs.


.. _session-item-object:

Kernel Session Item Object
--------------------------

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description
   * - ``id``
     - ``slug``
     - The kernel session ID.
   * - ``type``
     - ``str``
     - The kernel type (typically the name of runtime or programming lanauge).
   * - ``status``
     - ``enum[str]``
     - One of ``"preparing"``, ``"building``", ``"running"``, ``"restarting"``, ``"resizing"``, ``"success"``, ``"error"``, ``"terminating"``, ``"suspended"``.
   * - ``statusInfo``
     - ``str``
     - An optional message related to the current status. (e.g., error information)
   * - ``age``
     - ``int`` (msec)
     - The time elapsed since the kernel has started.
   * - ``execTime``
     - ``int`` (msec)
     - The time taken for execution. Excludes the time taken for being suspended, restarting, and resizing.
   * - ``numQueriesExecuted``
     - ``int``
     - The total number of queries executed after start-up.
   * - ``memoryUsed``
     - ``int`` (MiB)
     - The amount of memory currently used (sum of all resident-set size across instances). It may show a stale value.
   * - ``cpuUtil``
     - ``int`` (%)
     - The current CPU utilization (sum of all used cores across instances, hence may exceed 100%). It may show a stale value.

       .. versionchanged:: v3.20170615

          This had been separated into multiple credit-based fields, but that was never implemented properly.
          We has changed it to represent more intuitive value.

   * - ``config``
     - ``object``
     - :ref:`creation-config-object` specified when created.

.. _creation-config-object:

Creation Config Object
----------------------

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description

   * - ``environ``
     - ``object``
     - A dictionary object specifying additional environment variables.
       The values must be strings.

   * - ``mounts``
     - ``list[str]``
     - An optional list of the name of virtual folders that belongs to the current API key.
       These virtual folders are mounted under ``/home/work``.
       For example, if the virtual folder name is ``abc``, you can access it on
       ``/home/work/abc``.

       If the name contains a colon in the middle, the second part of the string indicates
       the alias location in the kernel's file system which is relative to ``/home/work``.

       You may mount up to 5 folders for each kernel session.

   * - ``clusterSize``
     - ``int``
     - The number of instances bundled for this session.

   * - ``instanceMemory``
     - ``int`` (MiB)
     - The maximum memory allowed per instance.
       The value is capped by the per-kernel image limit.
       Additional charges may apply on the public API service.

   * - ``instanceCores``
     - ``int``
     - The number of CPU cores.
       The value is capped by the per-kernel image limit.
       Additional charges may apply on the public API service.

   * - ``instanceGPUs``
     - ``float``
     - The fraction of GPU devices (1.0 means a whole device).
       The value is capped by the per-kernel image limit.
       Additional charges may apply on the public API service.

.. _vfolder-item-object:

Virtual Folder Item Object
--------------------------

.. list-table::
   :widths: 15 5 80
   :header-rows: 1

   * - Key
     - Type
     - Description
   * - ``name``
     - ``str``
     - The human readable name set when created.
   * - ``id``
     - ``slug``
     - The unique ID of the folder. Use this when making API requests referring this folder.
   * - ``linked``
     - ``bool``
     - Indicates if this folder is linked to an external service. (enterprise edition only)
   * - ``usedSize``
     - ``int`` (MiB)
     - The sum of the size of files in this folder.
   * - ``numFiles``
     - ``int``
     - The number of files in this folder.
   * - ``maxSize``
     - ``int`` (MiB)
     - The maximum size of this folder.
   * - ``created``
     - ``datetime``
     - The date and time when the folder is created.

