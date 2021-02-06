.. _linux-latest:

Building ROS 2 for QNX
=======================

.. contents:: Table of Contents
   :depth: 2
   :local:

Overview of the build process
-----------------------------

Starting with a QNX SDP7.1 installation along with the required cross compiled dependencies, the build process will cross compile ROS 2's source code against SDP7.1 and the cross compiled dependencies.
Binaries will be generated for the two architectures below:

aarch64le
x86_64

The generated files can then be transferred to the required target and used. The following document will go over the steps needed to cross compile the dependencies and ROS 2.

System requirements
-------------------

HOST:

- Ubuntu 20.04
- QNX SDP7.1

For instructions to install SDP7.1 please follow the link:

http://www.qnx.com/developers/docs/7.1/index.html#com.qnx.doc.qnxsdp.quickstart/topic/about.html

TARGET:

- A QNX supported architecture running QNX SDP7.1

System setup
------------

Set locale
^^^^^^^^^^
Make sure to set a locale that supports UTF-8.

The following is an example for setting locale.
However, it should be fine if you're using a different UTF-8 supported locale.

.. code-block:: bash

   sudo locale-gen en_US en_US.UTF-8
   sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
   export LANG=en_US.UTF-8

Add the ROS 2 apt repository
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. include:: ../_Apt-Repositories.rst

Install development tools and ROS tools
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

   sudo apt update && sudo apt install -y \
     build-essential \
     git \
     libbullet-dev \
     python3-colcon-common-extensions \
     python3-flake8 \
     python3-pip \
     python3-pytest-cov \
     python3-rosdep \
     python3-setuptools \
     python3-vcstool \
     wget \
     bc \
     subversion \
     autoconf \
     libtool-bin \
     libssl-dev \
     zlib1g-dev \
     rsync \
     rename
     
   # install some pip packages needed for testing
   python3 -m pip install -U \
     argcomplete \
     flake8-blind-except \
     flake8-builtins \
     flake8-class-newline \
     flake8-comprehensions \
     flake8-deprecated \
     flake8-docstrings \
     flake8-import-order \
     flake8-quotes \
     pytest-repeat \
     pytest-rerunfailures \
     pytest \
     setuptools \
     importlib-metadata \
     importlib-resources \
     Cython \
     numpy \
     lark-parser

.. code-block:: bash

   cd /opt && sudo wget https://cmake.org/files/v3.18/cmake-3.18.0-Linux-x86_64.sh
   sudo sh cmake-3.18.0-Linux-x86_64.sh --prefix=/opt && \
   sudo ln -s /opt/cmake-3.18.0-Linux-x86_64/bin/cmake /usr/local/bin/cmake
     
.. _Rolling_QNX-dev-get-ros2-code:

Get ROS 2 code
--------------

Create a workspace and clone all repos:

.. code-block:: bash

   mkdir -p ~/ros2_rolling/src
   cd ~/ros2_rolling
   wget https://raw.githubusercontent.com/ros2/ros2/master/ros2.repos
   vcs import src < ros2.repos

Building steps
--------------

1- From withing the directory ~/ros2_rolling, clone additional files necessary for building ROS 2 and the dependencies then merge them with your ROS 2 directory.

.. code-block:: bash

    cd ~/ros2_rolling
    git clone https://gitlab.com/qnx/ros2/ros2_qnx.git /tmp/ros2
    rsync -haz /tmp/ros2/* .
    rm -rf /tmp/ros2

2- Source qnxsdp-env.sh script.

.. code-block:: bash

    . ~/qnx710/qnxsdp-env.sh

Optional: Add the sourcing command to the end of ~/.bashrc if you would like the environment to be set every time for you.

3- Import the required QNX build files for each dependency by importing QNX dependencies repositories.

.. code-block:: bash

    mkdir -p src/qnx_deps
    vcs import src/qnx_deps < qnx_deps.repos

4- Run a script to automatically embed <build_depend> in the packages that depends on qnx_deps.

.. code-block:: bash

    ./check_deps.py --path=src -v

5- Before building ROS 2, some packages will need to be ignored first. Which are as following.

.. code-block:: bash

    ./colcon-ignore.sh

6- Export CPU variable according to your target architecture:

Please note: If no CPU is set all architectures are going to be built.

options for CPU: aarch64, x86_64

.. code-block:: bash
    
    export CPU=aarch64

7- Build ROS 2.

.. code-block:: bash

    ./build-ros2.sh

Setup your target
^^^^^^^^^^^^^^^^^

1- ssh to your target or run the following commands on your target directly.

2- make sure libffi is included with your image otherwise copy it over from your sdp

.. code-block:: bash
    
    scp ~/qnx710/target/qnx7/x86_64/usr/lib/libffi.so.6 root@<target_ip>:/usr/lib/
    ln -s /usr/lib/libffi.so.6 /usr/lib/libffi.so

3- Install pip on your target

.. code-block:: bash

    curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
    python get-pip.py

4- Install python dependencies on your target.

.. code-block:: bash
    
    pip install -U \
    colcon-common-extensions \
    importlib-metadata \
    importlib-resources \
    lark-parser

5- create a directory for ROS 2's installation.

.. code-block:: bash
    
    mkdir -p /opt/ros/rolling

6- Get the ip address of your target

.. code-block:: bash
    
    ifconfig

6- Check the amount of space available on your target and make sure you have enough space to copy the files over.

.. code-block:: bash

    df -h

7- Copy ROS 2 to your target.

Note: you will have to replace "your_target_architecture" with your target architecture.

On host:

.. code-block:: bash

    cd ~/ros2_rolling/install/<your_target_arch>/
    tar -czvf ros2_rolling.tar.gz *
    scp ros2_rolling.tar.gz root@target_ip_address:/opt/ros/rolling/

On target:

.. code-block:: bash

    cd /opt/ros/rolling
    tar -xzvf ros2_rolling.tar.gz

All the necessary files to run ROS 2 are now on your target.

Test the installation
^^^^^^^^^^^^^^^^^^^^^

1- ssh to your target and on one terminal run the following.

.. code-block:: bash

    export COLCON_CURRENT_PREFIX=/opt/ros/rolling
    . /opt/ros/rolling/local_setup.sh
    ros2 run demo_nodes_cpp talker
    
2- On another terminal run the following.

.. code-block:: bash

    export COLCON_CURRENT_PREFIX=/opt/ros/rolling
    . /opt/ros/rolling/local_setup.sh
    ros2 run demo_nodes_py listener
    
You should see the demos running on both terminals if the installation went successfull.

Developing your own code using ROS 2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now that we have ROS 2 binaries cross compiled along with the dependencies you can start building your own software against them.

Please use the toolchain file along with the build-ros2.sh script used to build ROS 2 with any of your future packages.

.. code-block:: bash

    mkdir -p ~/my_new_package/platform
    cp ~/ros2_rolling/platform/qnx.nto.toolchain.cmake ~/my_new_ros2_package/platform/
    cp ~/ros2_rolling/platform/build-ros2.sh ~/my_new_ros2_package/

Source your development environment which includes QNX environment and ROS2.

.. code-block:: bash

    . ~/qnx710/qnxsdp-env-ros2.sh
    . ~/ros2_rolling/install/<your_target_arch>/local_setup.bash

After you write your code and are ready to build you can run colcon by running the build-ros2.sh script.

.. code-block:: bash

    ./build-ros2.sh
