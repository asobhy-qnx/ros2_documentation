.. _linux-latest:

Building ROS 2 for QNX 
=======================

.. contents:: Table of Contents
   :depth: 2
   :local:


Overview of the build process
-----------------------------

Starting with a QNX SDP7.1 installation along with the required cross compiled dependencies, the build process will cross compile ROS 2's source code against SDP7.1 and the cross compiled dependencies.
Binaries will be generated for three architectures:

aarch64le
armv7-le
x86_64

The generated files can then be transferred to the required target and used. The following document will go over the steps needed to cross compile the dependencies and ROS 2.


System requirements
-------------------

HOST:

- A PC running with Ubuntu 18.04 or 20.04 and QNX SDP7.1

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

Install development tools and ROS tools
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Install tools needed for building dependencies

.. code-block:: bash

    sudo apt install -y \
     build-essential \
     bc \
     subversion \
     autoconf \
     libtool-bin \
     libssl-dev \
     zlib1g-dev \
     wget \
     git \
     rsync \
     rename

Install cmake 3.18.0

.. code-block:: bash
    
    sudo bash
    cd /opt
    wget https://cmake.org/files/v3.18/cmake-3.18.0-Linux-x86_64.sh
    sh cmake-3.18.0-Linux-x86_64.sh --prefix=/opt
    ln -s /opt/cmake-3.18.0-Linux-x86_64/bin/cmake /usr/local/bin/cmake
    exit
    
Check version.
    
.. code-block:: bash

    cmake --version

Install Python 3.8.0 dependencies

.. code-block:: bash

    sudo apt install -y \
     libsqlite3-dev \
     sqlite3 \
     bzip2 \
     libbz2-dev \
     zlib1g-dev \
     openssl \
     libgdbm-dev \
     libgdbm-compat-dev \
     liblzma-dev \
     libreadline-dev \
     libncursesw5-dev \
     libffi-dev \
     uuid-dev

Install Python 3.8.0

.. code-block:: bash

    cd /tmp
    wget https://www.python.org/ftp/python/3.8.0/Python-3.8.0.tgz
    tar -xf Python-3.8.0.tgz
    cd Python-3.8.0
    sudo ./configure --enable-optimizations
    sudo make -j$(nproc)
    sudo make altinstall
    sudo ln -s /usr/local/bin/python3.8 /usr/local/bin/python
    sudo ln -s /usr/local/bin/python3.8 /usr/local/bin/python3

Open a new terminal and verify Python version is correct.
    
.. code-block:: bash

    python --version
    python3 --version


Install standard ROS 2 development tools. Cython is needed for building numpy which is one of the dependencies needed to be built from source.

.. code-block:: bash

   sudo pip3.8 install \
    colcon-common-extensions \
    flake8 \
    pytest-cov \
    rosdep \
    setuptools \
    vcstool \
    lark-parser \
    numpy \
    Cython \
    importlib-metadata \
    importlib-resources
     
Install some packages needed for testing

.. code-block:: bash

   sudo pip3.8 install \
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
    setuptools

.. _Rolling_QNX-dev-get-ros2-code:

Get ROS 2 code
--------------

Create a workspace and clone all repos:

.. code-block:: bash

   mkdir -p ~/ros2_rolling/src
   cd ~/ros2_rolling
   wget https://raw.githubusercontent.com/ros2/ros2/master/ros2.repos
   vcs import src < ros2.repos


Build and Install dependencies
------------------------------

Important QNX keywords and a brief intro
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**QNX_HOST:**
Environment variable which provides the path to your ~/qnx_installation_path/host/operating_system_name/architecture/, e.g: ~/qnx710/host/linux/x86_64, and this is where your toolchain is.

**QNX_TARGET:**
Environment variable which provides the path to your ~/qnx_installation_path/target/qnx7, e.g: ~/qnx710/target/qnx7, and this is where the system root files for the three supported architectures exist

**QNX_STAGE:**
Environment variable which can be set by the user to provid the path to your cross compiled dependencies.

The environment variables above need to be set by a script before you start building for QNX. The script exists inside your SDP7.1 directory, e.g: qnx710/qnxsdp-env.sh.

Through out this document I will assume QNX SDP7.1 is installed under ~/qnx710 and will be referring to it as such. Please check any of the steps that I include ~/qnx710 in and change it according to your actual path.

You will need to source the script above before building for QNX, but first you need to do the following steps.


List of required dependencies
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The list below represent the dependencies required to be built from source, which is then followed by the build instructions.

Dashing, Foxy and Rolling dependencies:

.. code-block:: bash

    apr
    apr-util
    log4cxx
    asio
    eigen3
    libpng16
    opencv
    numpy --depends-on--> cython
    lxml --depends-on--> libxslt
    tinyxml2
    uncrustify

Foxy and Rolling extra dependencies:

.. code-block:: bash

    netifaces
    libbullet-dev
    memory


Dependencies build instructions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Setup your host
^^^^^^^^^^^^^^^

1- From withing the directory ~/ros2_rolling, clone additional files necessary for building ROS 2 and the dependencies then merge them with your ROS 2 directory.

.. code-block:: bash

    cd ~/ros2_rolling
    git clone https://github.com/asobhy-qnx/ros2.git /tmp/ros2
    rsync -haz /tmp/ros2/* .
    rm -rf /tmp/ros2


2- Create a staging directory which will contain the installation of your cross compiled dependencies.

.. code-block:: bash

    ./create-stage.sh

This will create the directory tree ~/ros2_rolling/qnx_stage

    
3- Create a second copy of the qnxsdp-env.sh script located in ~/qnx710, name it qnxsdp-env-ros2.sh and add the following to the end.

.. code-block:: bash

    cp ~/qnx710/qnxsdp-env.sh ~/qnx710/qnxsdp-env-ros2.sh; \
    echo -e "\nQNX_STAGE=$HOME/ros2_rolling/qnx_stage\nQCONF_OVERRIDE=$HOME/qnx710/qconf-override.mk\n\n \
    export QNX_STAGE QCONF_OVERRIDE\n\n \
    echo QNX_STAGE=\$QNX_STAGE\n \
    echo QCONF_OVERRIDE=\$QCONF_OVERRIDE" >> ~/qnx710/qnxsdp-env-ros2.sh


4- Under ~/qnx710 create a file named qconf-override.mk like so.

.. code-block:: bash

    echo -e "INSTALL_ROOT_nto := \$(QNX_STAGE)\nUSE_INSTALL_ROOT = 1" > ~/qnx710/qconf-override.mk

This will override the installation path of packages when you run "make install" to install files into your qnx_stage directory instead of into the sdp.


5- Source qnxsdp-env.sh script.

.. code-block:: bash

    . ~/qnx710/qnxsdp-env-ros2.sh

Optional: Add the sourcing command to the end of ~/.bashrc if you would like the environment to be set every time for you.

6- Import the required QNX build files for each dependency by importing QNX dependencies repositories.

.. code-block:: bash

    cd ~/ros2_rolling/qnx_deps
    mkdir src
    vcs import src < qnx_deps.repos


7- Build ROS 2 QNX dependencies. Please note this step will take quite sometime as it will clone, patch and build all the required dependencies

.. code-block:: bash

    ./build-deps.sh

Double check the installation of the dependencies in your staging directory ~/ros2_rolling/qnx_stage/usr/include and ~/ros2_rolling/qnx_stage/$CPUVARDIR/usr/lib


8- After the dependencies are built and installed successfully you can start building ROS 2 but some packages will need to be ignored first. Which are as following.

.. code-block:: bash

    Visualization
    Uncrustify
    CycloneDDS
    Mimick
    Rttest
    Pendulum Control Demo


Run the script colcon-ignore.sh and it will add COLCON_IGNORE to all the packages above to prevent them from being built.

.. code-block:: bash

    cd ~/ros2_rolling
    ./colcon-ignore.sh


9- Build ROS 2.

.. code-block:: bash

    ./build-ros2.sh


Setup your target
^^^^^^^^^^^^^^^^^

1- ssh to your target or run the following commands on your target directly.


2- make sure libffi is included with your image otherwise copy it over from your sdp

.. code-block:: bash
    
    rsync -havz ~/qnx710/target/qnx7/x86_64/usr/lib/libffi.* root@target_ip:/usr/lib/


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


7- Copy the dependencies to your target.

Note: you will have to replace "target_ip_address" with your target ip address.

.. code-block:: bash

On host:

.. code-block:: bash

    cd ~/ros2_rolling/qnx_stage/x86_64/usr/
    tar -czvf qnxdeps.tar.gz *
    scp qnxdeps.tar.gz root@target_ip_address:/usr/

On target:

.. code-block:: bash

    cd /usr
    tar -xzvf qnxdeps.tar.gz

8- Copy ROS 2 to your target.

Note: you will have to replace "your_target_architecture" with your target architecture.

On host:

.. code-block:: bash

    cd ~/ros2_rolling/install/x86_64/
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
    . ~/ros2_rolling/install/your_target_arch/local_setup.bash


After you write your code and are ready to build you can run colcon by running the build-ros2.sh script.

.. code-block:: bash

    ./build-ros2.sh
