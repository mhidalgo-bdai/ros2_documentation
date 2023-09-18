Launch
======

.. contents:: Table of Contents
   :local:

Overview
--------

A ROS 2 system typically consists of multiple :doc:`nodes <About-Nodes>` running across different processes (and even different machines).
While it is possible to start each of these nodes manually, it can get cumbersome rather quickly.
Automation is also non-trivial, even if existing tools for generic process control and configuration management are leveraged, let alone crafting it from first principles (e.g. proper OS process management using shell scripts or general purpose system programming languages can be tricky on its own).

``launch`` provides the means to orchestrate the execution of ROS 2 systems, in ROS 2 terms.
_Launch files_ describe the orchestration procedure, including but not limited to which nodes to run, how to run them, and which arguments to use for them.
These descriptions are then executed, starting processes, monitoring them, reporting on their state, and even reacting to their behavior.

Moving parts
------------

``launch`` works with _actions_, abstract representations of computational procedures with side effects on its execution environment. Logging to standard output, executing a program, shutting down ``launch`` itself, are examples of actions. An action may also encompass another action or a collection thereof to describe more complex functionality. A collection of actions makes up the _description_ of a procedure that ``launch`` can take and execute. These descriptions are typically read from source files, so called _launch files_, which may be written in Python, or using specific XML or YAML syntax.

While the extent of the execution environment of an action depends on its nature e.g. logging to standard output is circumscribed to the ``launch`` process whereas executing a program reaches out to the host operating system, ``launch`` maintains an execution _context_ in which these actions take place and through which these can interact between them and with the user.
The ``launch`` context holds _configuration variables_ (i.e. state) and propagates _events_, both of which are available to actions and to ``launch`` itself.
Events are signals that originate internally in actions or externally on user input. It is through events, for example, that ``launch`` description execution (i.e. inclusion) is triggered.
Configuration variables populate the ``launch`` context. It is through configuration variables, for example, that ``launch`` descriptions can take arguments before execution.

In connection with its context, ``launch`` defines _conditions_, _substitutions_, and _event handlers_ to broaden its scope of application.
The first two are different forms of expressions evaluated in ``launch`` context (though they may also tap into the larger execution environment): conditions are boolean predicates, used to define conditional actions, while substitutions are string interpolation expressions that enable dynamic ``launch`` descriptions.
Event handlers, on the other hand, are collections of actions registered for execution when and if the corresponding event occurs.

As it may be apparent by now, ``launch`` descriptions are essentially programs in a domain specific language tailored for process orchestration, and, in particular, ROS 2 system orchestration. When composed using the building blocks available in its native Python implementation, these descriptions resemble `ASTs <https://en.wikipedia.org/wiki/Abstract_syntax_tree>`_ in procedural programming languages. The analogy has its limits: context is not implicitly restricted to syntactical boundaries like it would for typical variable scopes, and action execution is naturally concurrent as opposed to sequential, to name a few. However, it does bring about an important distinction that is easy to miss when writing launch files in Python: no action nor condition nor substitution carries out a computation upon instantiation but simply specifies a computation to be carried out in runtime.

.. note::

    A simple mental model to reason about Python launch files is one of two phases: a configuration phase, during which the description is fully constructed, and an execution phase, during which ``launch`` executes based on the provided description until shutdown.

Practical aspects
-----------------

* Launch files can be written in Python, XML, or YAML.
  XML and YAML launch files keep it simple and avoid the confusion that a domain specific language embedded in a general purpose programming language may bring, but the flexibility and complexity that Python launch files afford may sometimes be useful if not necessary.
  Refer to the examples in the :doc:`../../How-To-Guides/Launch-file-different-formats` guide on how to write launch files.
* Launch files can include other launch files, written in any of the supported languages, for modular system description and component reuse.
* Launch files, written in any of the supported languages, can be run using the ``ros2 launch`` command, which also can take their arguments.
  Note the difference with the ``ros2 run`` command, which works with executables installed by ROS 2 packages, not launch files.
* Most of ``launch`` infrastructure lives in the ``launch`` Python package, but ROS 2 specifics live in the ``launch_ros`` Python package.

References
----------

The most thorough reference on the design of ``launch`` is, unsurprisingly, its seminal `design document <https://design.ros2.org/articles/roslaunch.html>`__ (which even includes functionality not yet available).
[``launch`` documentation](https://docs.ros.org/en/rolling/p/launch`` complements it, detailing the architecture of the core Python library.
For everything else, both ``launch`` and ``launch_ros`` APIs are documented.
