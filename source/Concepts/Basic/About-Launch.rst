Launch
======

.. contents:: Table of Contents
   :local:

Overview
--------

A ROS 2 system typically consists of multiple nodes running across different processes (and even different machines).
While it is possible to run each of these nodes separately, it can get cumbersome rather quickly.

The purpose of the ``launch`` system in ROS 2 is to provide the means to orchestrate the execution of such a system. This tool uses _launch files_ to describe the system and its configuration, but not limited to which programs to run, how to run them and which arguments to use for them. These descriptions are then executed, starting processes, monitoring them, reporting on their state, and even reacting to their behavior.

System descriptions
-------------------

``launch`` works with _descriptions_, which, for the most part, amount to sequences of _actions_ executing in a _context_ or scope. All actions have side effects, which may be limited to the context in which they are running or extend to the larger execution environment (e.g. the local operating system). These side effects will typically outlive the context in which the corresponding action was run, which may be obvious for an action that changes an environment variable but not for an action that executes a process. Actions may also generate _events_, signals that are propagated through the context, that may be handled by _event handlers_.

Actions alone allow for static descriptions that act upon and react to the environment in predefined ways. For a description to adapt to external input, _conditions_ and _substitutions_ exist. The former are boolean predicates, used for conditional actions, while the latter resemble text macros, useful for dynamic action configuration.

In a way, ``launch`` descriptions are not that different from ASTs in procedural programming languages. The analogy brings about an important distinction that is easy to miss when writing launch files using the native Python APIs: neither actions nor conditions nor substitutions are computations but descriptions of computations to be carried out on launch. That is how actions can be implemented as sequences of other actions -- launch file inclusion being the paradigmatic example of this.

For day to day use, a simple mental model to reason about these descriptions is one of two phases: a configuration phase, up until the description is fully constructed, and an execution phase, during which the description is acted on and until ``launch`` shuts down.

Practical aspects
-----------------

Launch files can be written in Python, XML, or YAML, and run using the ``ros2 launch`` command. Launch files can include other launch files, written in any of the supported languages, for modular system description and component reuse. Context in launch is in general not restricted to syntactical boundaries (e.g. files, groups) -- scoping is an explicit operation (see push / pop primitives), making variable propagation across launch file hierarchies the default. Launch files can take arguments which are put in context, making them available to substitutions.

Check the :doc:`../../How-To-Guides/Launch-file-different-formats` guide on how to write launch files.

References
----------

The most thorough reference on the design of ``launch`` is, unsurprisingly, its seminal `design document <https://design.ros2.org/articles/roslaunch.html>`__ (which even includes functionality not yet available). [``launch`` documentation](https://docs.ros.org/en/rolling/p/launch`` complements it, detailing the architecture of the core Python library. For everything else, both ``launch`` and ``launch_ros`` APIs are documented.
