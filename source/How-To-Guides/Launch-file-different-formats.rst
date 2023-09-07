.. redirect-from::

  Guides/Launch-file-different-formats

Using Python, XML, and YAML for ROS 2 Launch Files
==================================================

.. contents:: Table of Contents
   :depth: 1
   :local:

ROS 2 launch files can be written in Python, XML, and YAML.
This guide shows how to use these different formats to accomplish the same task. It also discusses when to use each format.

Examples
--------

Taking arguments
^^^^^^^^^^^^^^^^

Each launch file performs the following actions:

* Declares arguments with and without defaults.
* Logs a message that interpolates those arguments.

.. tabs::

   .. group-tab:: Python

      .. code-block:: python

        # example_launch.py

        from launch import LaunchDescription
        from launch.actions import DeclareLaunchArgument
        from launch.actions import LogInfo
        from launch.substitutions import LaunchConfiguration

        def generate_launch_description():
            return LaunchDescription([
                DeclareLaunchArgument('who'),
                DeclareLaunchArgument('where', default_value='home'),

                LogInfo(msg=[LaunchConfiguration('who'), ' is at ', LaunchConfiguration('where')])
            ])


   .. group-tab:: XML

      .. code-block:: xml

        <!-- example_launch.xml -->

        <launch>
            <arg name="who"/>
            <arg name="where" default="home"/>

            <log message="$(var who) is at $(var where)"/>
        </launch>

   .. group-tab:: YAML

      .. code-block:: yaml

        # example_launch.yaml

        launch:
        - arg:
            name: "who"
        - arg:
            name: "where"
            default: "home"
        - log:
            message: "$(var who) is at $(var where)"

Requiring processes
^^^^^^^^^^^^^^^^^^^

Each launch file performs the following actions:

* Declares an argument with defaults.
* Sleeps for some time, then shuts down launch.

.. tabs::

   .. group-tab:: Python

      .. code-block:: python

        # example_launch.py

        from launch import LaunchDescription
        from launch.actions import DeclareLaunchArgument
        from launch.actions import ExecuteProcess
        from launch.actions import LogInfo
        from launch.actions import Shutdown
        from launch.substitutions import LaunchConfiguration

        def generate_launch_description():
            return LaunchDescription([
                DeclareLaunchArgument('duration', default_value='10'),
                ExecuteProcess(
                    cmd=['sleep', LaunchConfiguration('duration')],
                    on_exit=[
                        LogInfo(msg='Done sleeping!'),
                        Shutdown(reason='main process terminated')
                    ]
                ),
                LogInfo(msg=['Sleeping for ', LaunchConfiguration('duration'), ' seconds']),
            ])

   .. group-tab:: XML

      .. code-block:: xml

        <!-- Currently unsupported -->

   .. group-tab:: YAML

      .. code-block:: yaml

        # Currently unsupported

Replicating hierarchies
^^^^^^^^^^^^^^^^^^^^^^^

Each launch file performs the following actions:

* Declares an argument without defaults.
* Generates namespaced groups iteratively based on that argument.

.. tabs::

   .. group-tab:: Python

      .. code-block:: python

        # example_launch.py

        from launch import LaunchDescription
        from launch.actions import DeclareLaunchArgument
        from launch.actions import GroupAction
        from launch.actions import OpaqueFunction
        from launch.substitutions import LaunchConfiguration
        from launch_ros.actions import Node
        from launch_ros.actions import PushRosNamespace

        def generate_turtles_description(context, turtles):
            return [
                GroupAction(
                    actions=[
                        PushRosNamespace(turtle_name),
                        Node(
                            package='turtlesim',
                            executable='turtlesim_node',
                            output='screen')
                    ]
                )
                for turtle_name in turtles.perform(context).split()
            ]

        def generate_launch_description():
            return LaunchDescription([
                DeclareLaunchArgument('turtles'),
                OpaqueFunction(
                    function=generate_turtles_description,
                    args=[LaunchConfiguration('turtles')])
            ])

   .. group-tab:: XML

      .. code-block:: xml

        <!-- Currently unsupported -->

   .. group-tab:: YAML

      .. code-block:: yaml

        # Currently unsupported

Cleaning after
^^^^^^^^^^^^^^

Each launch file performs the following actions:

* Declares an argument without defaults.
* Registers an action to execute on shutdown.
* Forces shutdown after a time specified by the argument.

.. tabs::

   .. group-tab:: Python

      .. code-block:: python

        # example_launch.py

        from launch import LaunchDescription
        from launch.actions import DeclareLaunchArgument
        from launch.actions import ExecuteProcess
        from launch.actions import RegisterEventHandler
        from launch.actions import Shutdown
        from launch.actions import TimerAction
        from launch.event_handlers import OnShutdown
        from launch.substitutions import LaunchConfiguration

        def generate_launch_description():
            return LaunchDescription([
                DeclareLaunchArgument('timeout'),
                RegisterEventHandler(OnShutdown(on_shutdown=[
                    ExecuteProcess(cmd=['rm', '-f', '/tmp/resource']),
                ])),
                ExecuteProcess(cmd=['touch', '/tmp/resource']),
                TimerAction(period=LaunchConfiguration('timeout'), actions=[
                    Shutdown(reason='launch timed out!')
                ])
            ])

   .. group-tab:: XML

      .. code-block:: xml

        <!-- Currently unsupported -->

   .. group-tab:: YAML

      .. code-block:: yaml

        # Currently unsupported

Managing lifecycles
^^^^^^^^^^^^^^^^^^^

Each launch file performs the following actions:

* Registers a custom event handler to manage lifecycles synchronously.
* Starts talker and listener managed nodes.

.. tabs::

   .. group-tab:: Python

      .. code-block:: python

        # example_launch.py

        from launch import LaunchDescription
        from launch.actions import RegisterEventHandler
        from launch.actions import EmitEvent
        from launch.event_handler import BaseEventHandler
        from launch.events import Shutdown
        from launch.events import matches_action
        from launch.events.process import ProcessStarted
        from launch_ros.actions import LifecycleNode
        from launch_ros.events.lifecycle import ChangeState
        from launch_ros.events.lifecycle import StateTransition

        import lifecycle_msgs.msg

        class LifecycleManager(BaseEventHandler):

            BRINGUP_SEQUENCE = {
                lifecycle_msgs.msg.State.PRIMARY_STATE_UNCONFIGURED: lifecycle_msgs.msg.Transition.TRANSITION_CONFIGURE,
                lifecycle_msgs.msg.State.PRIMARY_STATE_INACTIVE: lifecycle_msgs.msg.Transition.TRANSITION_ACTIVATE
            }

            SHUTDOWN_SEQUENCE = {
                lifecycle_msgs.msg.State.PRIMARY_STATE_UNCONFIGURED: lifecycle_msgs.msg.Transition.TRANSITION_UNCONFIGURED_SHUTDOWN,
                lifecycle_msgs.msg.State.PRIMARY_STATE_INACTIVE: lifecycle_msgs.msg.Transition.TRANSITION_INACTIVE_SHUTDOWN,
                lifecycle_msgs.msg.State.PRIMARY_STATE_ACTIVE: lifecycle_msgs.msg.Transition.TRANSITION_ACTIVE_SHUTDOWN
            }

            def __init__(self, nodes):
                self.managees = {node: lifecycle_msgs.msg.State.PRIMARY_STATE_UNKNOWN for node in nodes}
                matcher = (lambda event: (
                    isinstance(event, (ProcessStarted, StateTransition)) and event.action in self.managees))
                super().__init__(matcher=matcher)

            def handle(self, event, context):
                if isinstance(event, ProcessStarted):
                    self.managees[event.action] = lifecycle_msgs.msg.State.PRIMARY_STATE_UNCONFIGURED
                if isinstance(event, StateTransition):
                    self.managees[event.action] = event.msg.goal_state.id
                states = list(set(self.managees.values()))
                common_state = states[0] if len(states) == 1 else None
                if common_state in self.BRINGUP_SEQUENCE:
                    matcher = lambda action: action in self.managees
                    return [
                        EmitEvent(event=ChangeState(
                            lifecycle_node_matcher=matcher,
                            transition_id=self.BRINGUP_SEQUENCE[common_state]))]
                return None

        def generate_launch_description():
            first_talker_node = LifecycleNode(
                package='lifecycle', executable='lifecycle_talker',
                name='first_talker', namespace='', output='screen')
            second_talker_node = LifecycleNode(
                package='lifecycle', executable='lifecycle_talker',
                name='second_talker', namespace='', output='screen')
            listener_node = LifecycleNode(
                package='lifecycle', executable='lifecycle_listener',
                name='listener', namespace='', output='screen',
                remappings=[('/lc_talker/transition_event',
                             '/first_talker/transition_event')])
            manager = LifecycleManager([first_talker_node, second_talker_node])
            return LaunchDescription([
                RegisterEventHandler(manager),
                first_talker_node, second_talker_node, listener_node
            ])

   .. group-tab:: XML

      .. code-block:: xml

        <!-- Currently unsupported -->

   .. group-tab:: YAML

      .. code-block:: yaml

        # Currently unsupported

Launching many nodes
^^^^^^^^^^^^^^^^^^^^

Each launch file performs the following actions:

* Setup command line arguments with defaults
* Include another launch file
* Include another launch file in another namespace
* Start a node and setting its namespace
* Start a node, setting its namespace, and setting parameters in that node (using the args)
* Create a node to remap messages from one topic to another

.. tabs::

   .. group-tab:: Python

      .. code-block:: python

        # example_launch.py

        import os

        from ament_index_python import get_package_share_directory

        from launch import LaunchDescription
        from launch.actions import DeclareLaunchArgument
        from launch.actions import GroupAction
        from launch.actions import IncludeLaunchDescription
        from launch.launch_description_sources import PythonLaunchDescriptionSource
        from launch.substitutions import LaunchConfiguration
        from launch.substitutions import TextSubstitution
        from launch_ros.actions import Node
        from launch_ros.actions import PushRosNamespace
        from launch_xml.launch_description_sources import XMLLaunchDescriptionSource
        from launch_yaml.launch_description_sources import YAMLLaunchDescriptionSource


        def generate_launch_description():

            # args that can be set from the command line or a default will be used
            background_r_launch_arg = DeclareLaunchArgument(
                "background_r", default_value=TextSubstitution(text="0")
            )
            background_g_launch_arg = DeclareLaunchArgument(
                "background_g", default_value=TextSubstitution(text="255")
            )
            background_b_launch_arg = DeclareLaunchArgument(
                "background_b", default_value=TextSubstitution(text="0")
            )
            chatter_py_ns_launch_arg = DeclareLaunchArgument(
                "chatter_py_ns", default_value=TextSubstitution(text="chatter/py/ns")
            )
            chatter_xml_ns_launch_arg = DeclareLaunchArgument(
                "chatter_xml_ns", default_value=TextSubstitution(text="chatter/xml/ns")
            )
            chatter_yaml_ns_launch_arg = DeclareLaunchArgument(
                "chatter_yaml_ns", default_value=TextSubstitution(text="chatter/yaml/ns")
            )

            # include another launch file
            launch_include = IncludeLaunchDescription(
                PythonLaunchDescriptionSource(
                    os.path.join(
                        get_package_share_directory('demo_nodes_cpp'),
                        'launch/topics/talker_listener_launch.py'))
            )
            # include a Python launch file in the chatter_py_ns namespace
            launch_py_include_with_namespace = GroupAction(
                actions=[
                    # push_ros_namespace to set namespace of included nodes
                    PushRosNamespace('chatter_py_ns'),
                    IncludeLaunchDescription(
                        PythonLaunchDescriptionSource(
                            os.path.join(
                                get_package_share_directory('demo_nodes_cpp'),
                                'launch/topics/talker_listener_launch.py'))
                    ),
                ]
            )

            # include a xml launch file in the chatter_xml_ns namespace
            launch_xml_include_with_namespace = GroupAction(
                actions=[
                    # push_ros_namespace to set namespace of included nodes
                    PushRosNamespace('chatter_xml_ns'),
                    IncludeLaunchDescription(
                        XMLLaunchDescriptionSource(
                            os.path.join(
                                get_package_share_directory('demo_nodes_cpp'),
                                'launch/topics/talker_listener_launch.xml'))
                    ),
                ]
            )

            # include a yaml launch file in the chatter_yaml_ns namespace
            launch_yaml_include_with_namespace = GroupAction(
                actions=[
                    # push_ros_namespace to set namespace of included nodes
                    PushRosNamespace('chatter_yaml_ns'),
                    IncludeLaunchDescription(
                        YAMLLaunchDescriptionSource(
                            os.path.join(
                                get_package_share_directory('demo_nodes_cpp'),
                                'launch/topics/talker_listener_launch.yaml'))
                    ),
                ]
            )

            # start a turtlesim_node in the turtlesim1 namespace
            turtlesim_node = Node(
                package='turtlesim',
                namespace='turtlesim1',
                executable='turtlesim_node',
                name='sim'
            )

            # start another turtlesim_node in the turtlesim2 namespace
            # and use args to set parameters
            turtlesim_node_with_parameters = Node(
                package='turtlesim',
                namespace='turtlesim2',
                executable='turtlesim_node',
                name='sim',
                parameters=[{
                    "background_r": LaunchConfiguration('background_r'),
                    "background_g": LaunchConfiguration('background_g'),
                    "background_b": LaunchConfiguration('background_b'),
                }]
            )

            # perform remap so both turtles listen to the same command topic
            forward_turtlesim_commands_to_second_turtlesim_node = Node(
                package='turtlesim',
                executable='mimic',
                name='mimic',
                remappings=[
                    ('/input/pose', '/turtlesim1/turtle1/pose'),
                    ('/output/cmd_vel', '/turtlesim2/turtle1/cmd_vel'),
                ]
            )

            return LaunchDescription([
                background_r_launch_arg,
                background_g_launch_arg,
                background_b_launch_arg,
                chatter_py_ns_launch_arg,
                chatter_xml_ns_launch_arg,
                chatter_yaml_ns_launch_arg,
                launch_include,
                launch_py_include_with_namespace,
                launch_xml_include_with_namespace,
                launch_yaml_include_with_namespace,
                turtlesim_node,
                turtlesim_node_with_parameters,
                forward_turtlesim_commands_to_second_turtlesim_node,
            ])


   .. group-tab:: XML

      .. code-block:: xml

        <!-- example_launch.xml -->

        <launch>

            <!-- args that can be set from the command line or a default will be used -->
            <arg name="background_r" default="0" />
            <arg name="background_g" default="255" />
            <arg name="background_b" default="0" />
            <arg name="chatter_py_ns" default="chatter/py/ns" />
            <arg name="chatter_xml_ns" default="chatter/xml/ns" />
            <arg name="chatter_yaml_ns" default="chatter/yaml/ns" />

            <!-- include another launch file -->
            <include file="$(find-pkg-share demo_nodes_cpp)/launch/topics/talker_listener_launch.py" />
            <!-- include a Python launch file in the chatter_py_ns namespace-->
            <group>
                <!-- push_ros_namespace to set namespace of included nodes -->
                <push_ros_namespace namespace="$(var chatter_py_ns)" />
                <include file="$(find-pkg-share demo_nodes_cpp)/launch/topics/talker_listener_launch.py" />
            </group>
            <!-- include a xml launch file in the chatter_xml_ns namespace-->
            <group>
                <!-- push_ros_namespace to set namespace of included nodes -->
                <push_ros_namespace namespace="$(var chatter_xml_ns)" />
                <include file="$(find-pkg-share demo_nodes_cpp)/launch/topics/talker_listener_launch.xml" />
            </group>
            <!-- include a yaml launch file in the chatter_yaml_ns namespace-->
            <group>
                <!-- push_ros_namespace to set namespace of included nodes -->
                <push_ros_namespace namespace="$(var chatter_yaml_ns)" />
                <include file="$(find-pkg-share demo_nodes_cpp)/launch/topics/talker_listener_launch.yaml" />
            </group>

            <!-- start a turtlesim_node in the turtlesim1 namespace -->
            <node pkg="turtlesim" exec="turtlesim_node" name="sim" namespace="turtlesim1" />
            <!-- start another turtlesim_node in the turtlesim2 namespace
                and use args to set parameters -->
            <node pkg="turtlesim" exec="turtlesim_node" name="sim" namespace="turtlesim2">
                <param name="background_r" value="$(var background_r)" />
                <param name="background_g" value="$(var background_g)" />
                <param name="background_b" value="$(var background_b)" />
            </node>
            <!-- perform remap so both turtles listen to the same command topic -->
            <node pkg="turtlesim" exec="mimic" name="mimic">
                <remap from="/input/pose" to="/turtlesim1/turtle1/pose" />
                <remap from="/output/cmd_vel" to="/turtlesim2/turtle1/cmd_vel" />
            </node>
        </launch>

   .. group-tab:: YAML

      .. code-block:: yaml

        # example_launch.yaml

        launch:

        # args that can be set from the command line or a default will be used
        - arg:
            name: "background_r"
            default: "0"
        - arg:
            name: "background_g"
            default: "255"
        - arg:
            name: "background_b"
            default: "0"
        - arg:
            name: "chatter_py_ns"
            default: "chatter/py/ns"
        - arg:
            name: "chatter_xml_ns"
            default: "chatter/xml/ns"
        - arg:
            name: "chatter_yaml_ns"
            default: "chatter/yaml/ns"


        # include another launch file
        - include:
            file: "$(find-pkg-share demo_nodes_cpp)/launch/topics/talker_listener_launch.py"

        # include a Python launch file in the chatter_py_ns namespace
        - group:
            - push_ros_namespace:
                namespace: "$(var chatter_py_ns)"
            - include:
                file: "$(find-pkg-share demo_nodes_cpp)/launch/topics/talker_listener_launch.py"

        # include a xml launch file in the chatter_xml_ns namespace
        - group:
            - push_ros_namespace:
                namespace: "$(var chatter_xml_ns)"
            - include:
                file: "$(find-pkg-share demo_nodes_cpp)/launch/topics/talker_listener_launch.xml"

        # include a yaml launch file in the chatter_yaml_ns namespace
        - group:
            - push_ros_namespace:
                namespace: "$(var chatter_yaml_ns)"
            - include:
                file: "$(find-pkg-share demo_nodes_cpp)/launch/topics/talker_listener_launch.yaml"

        # start a turtlesim_node in the turtlesim1 namespace
        - node:
            pkg: "turtlesim"
            exec: "turtlesim_node"
            name: "sim"
            namespace: "turtlesim1"

        # start another turtlesim_node in the turtlesim2 namespace and use args to set parameters
        - node:
            pkg: "turtlesim"
            exec: "turtlesim_node"
            name: "sim"
            namespace: "turtlesim2"
            param:
            -
              name: "background_r"
              value: "$(var background_r)"
            -
              name: "background_g"
              value: "$(var background_g)"
            -
              name: "background_b"
              value: "$(var background_b)"

        # perform remap so both turtles listen to the same command topic
        - node:
            pkg: "turtlesim"
            exec: "mimic"
            name: "mimic"
            remap:
            -
                from: "/input/pose"
                to: "/turtlesim1/turtle1/pose"
            -
                from: "/output/cmd_vel"
                to: "/turtlesim2/turtle1/cmd_vel"

Using launch files from the command line
--------------------------------------------

Launching
^^^^^^^^^

Any of the launch files above can be run with ``ros2 launch``.
To try them locally, you can either create a new package and use

.. code-block:: console

  ros2 launch <package_name> <launch_file_name>

or run the file directly by specifying the path to the launch file

.. code-block:: console

  ros2 launch <path_to_launch_file>

Setting arguments
^^^^^^^^^^^^^^^^^

To set the arguments that are passed to the launch file, you should use ``key:=value`` syntax.
For example, you can set the value of ``background_r`` in the following way:

.. code-block:: console

  ros2 launch <package_name> <launch_file_name> background_r:=255

or

.. code-block:: console

  ros2 launch <path_to_launch_file> background_r:=255

Controlling the turtles
^^^^^^^^^^^^^^^^^^^^^^^

To test that the remapping is working, you can control the turtles by running the following command in another terminal:

.. code-block:: console

  ros2 run turtlesim turtle_teleop_key --ros-args --remap __ns:=/turtlesim1


Python, XML, or YAML: Which should I use?
-----------------------------------------

.. note::

  Launch files in ROS 1 were written in XML, so XML may be the most familiar to people coming from ROS 1.
  To see what's changed, you can visit :doc:`Migrating-from-ROS1/Migrating-Launch-Files`.

For most applications the choice of which ROS 2 launch format comes down to developer preference.
However, if your launch file requires flexibility that you cannot achieve with XML or YAML, you can use Python to write your launch file.
Using Python for ROS 2 launch is more flexible because:

* Python is a scripting language, and thus you can leverage the language and its libraries in your launch files.
* `ros2/launch <https://github.com/ros2/launch>`_ (general launch features) and `ros2/launch_ros <https://github.com/ros2/launch_ros>`_ (ROS 2 specific launch features) are written in Python and thus you have lower level access to launch features that may not be yet exposed via XML and YAML.

That being said, a launch file written in Python may be more complex and verbose than one in XML or YAML.
