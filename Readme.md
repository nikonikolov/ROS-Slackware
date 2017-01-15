# Install ROS Kinetic on Slackware 14.2

This guide has been written and tested for Slackware 14.2 and a ROS Kinetic Desktop Install. If you 
have a different version of Slackware, steps might be slightly different.

Some other useful references are:
* Install ROS from source: http://wiki.ros.org/Installation/Source
* Another guide for installing on Slackware 14.2: http://barelywalking.com/?p=13
* A guide for installing on Slackware 14.1: http://barelywalking.com/?p=5

Note: ROS still primarily uses Python 2.7 and you should use that version of python for the installation.

##1. Prerequisites

###1.1 Install pip if you don't already have it
```
wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
```

###1.2 Install ros tools

```
sudo pip install -U rosdep rosinstall rosinstall_generator wstool
```

###1.3 Initialize rosdep 
```
sudo rosdep init
rosdep update
```


##2. Installing dependencies

###2.1 Install sbotools
If you already have sbotools go to the next step

The Slackware installer that ROS uses is sbotools. This installer automates the process of installing
packages from SlackBuilds.org - a third party repository for Slackware Linux. If you don't wish to isntall
sbotools you will have to manually install all dependencies.

To install sbotools:

```
wget https://pink-mist.github.io/sbotools/downloads/sbotools-2.0.tar.gz
sudo installpkg sbotools-2.0.tar.gz
```

###2.2 Update your list of packages

Make sure to execute this step as some of the dependencies were submitted on SlackBuilds.org very recently.
```
sudo sbocheck
```

###2.3 Install some of the dependencies

We are going to preinstall some of the ROS dependencies because they are more special.

####2.3.1 Install VTK

VTK requires to be built with JAVA support which needs to be set manually.
First we need to install `jdk`. Installing it directly from sbotools fails because it cannot automatically
download the source from Oracle's website. Go to https://slackbuilds.org/repository/14.2/development/jdk/
and install jdk manually. Make sure to select the right architecture (32 or 64-bit).

After you are done, make sure to **reopen terminal** - we need to update environment variables that jdk set.
Otherwise VTK installation will fail. We are now ready to install VTK:

```
sudo sboinstall VTK
```

When asked whether you want to set options when building VTK, say yes and pass the option:

```
JAVA=yes
```

####2.3.2 Install log4cxx

log4cxx is not available on SlackBuilds.org so we need to install it manually. It is available as part of this
repository:

```
git clone https://github.com/nikonikolov/ROS-Slackware.git
cd ROS-Slackware/log4cxx
sudo ./log4cxx.SlackBuild
sudo installpkg log4cxx-0.10.0-x86_64-1root.txz
```

####2.3.3 Install urdfdom

Although this package is available on SlackBuilds.org we still need to manually beforehand due to a strange way that sbotools handles dependencies:

```
sudo sboinstall urdfdom_headers
sudo sboinstall urdfdom
```


##3. Initialize rosdep
```
mkdir ~/ros_catkin_ws
cd ~/ros_catkin_ws
```

####Desktop-Full Install: 
*ROS, rqt, rviz, robot-generic libraries, 2D/3D simulators, navigation and 2D/3D perception* 

This guide has been tested for this installation.

```
rosinstall_generator desktop_full --rosdistro kinetic --deps --wet-only --tar > kinetic-desktop-full-wet.rosinstall
wstool init -j8 src kinetic-desktop-full-wet.rosinstall
```


####Desktop Install (recommended): 
*ROS, rqt, rviz, and robot-generic libraries* 

This guide has been tested for this installation.

```
rosinstall_generator desktop --rosdistro kinetic --deps --wet-only --tar > kinetic-desktop-wet.rosinstall
wstool init -j8 src kinetic-desktop-wet.rosinstall
```

####ROS-Comm: 
*(Bare Bones) ROS package, build, and communication libraries. No GUI tools.*

Although not tested for this kind of installation, this guide should work for it as all dependencies should be satisfied.

```
rosinstall_generator desktop --rosdistro kinetic --deps --wet-only --tar > kinetic-desktop-wet.rosinstall
wstool init -j8 src kinetic-desktop-wet.rosinstall
```

##4. Install remaining dependencies
We should now install the rest of the dependencies for Slackware. We can do that using rosdep which will 
automatically install the missing dependencies using sbotools and pip.

```
rosdep install --from-paths src --ignore-src --rosdistro kinetic -y
```

Now wait for ages until it is done.

If you don't want to use sbotools and pip to install dependencies, instead of the above you can execute

```
rosdep install --from-paths src --ignore-src --rosdistro kinetic -y -s
```

This will output a list of commands that contains that rosdep would have otherwise executed. In that list you
can find the dependencies that you are missing. It will look like:

```
#[sbotools] Installation commands:
  sudo -H sboinstall -r protobuf
  sudo -H sboinstall -r sbcl
#[pip] Installation commands:
  sudo -H pip install -U empy
  ...
```

##5. Build the catkin Workspace

Now that we have satisfied all dependencies we are almost ready to go. There are a few issues we need to fix
before building. 


###5.1 Set PYTHONPATH


Execute these commands in your current terminal:

```
PYTHONPATH=$PYTHONPATH:~/ros_catkin_ws/src/catkin/python/catkin
PYTHONPATH=$PYTHONPATH:~/ros_catkin_ws/install_isolated/lib64/python2.7/site-packages
PYTHONPATH=$PYTHONPATH:~/ros_catkin_ws/install_isolated/lib/python2.7/site-packages
export PYTHONPATH
```

###5.2 Fix build issues

If we try to build the workspace now, we will run in build erros due to Slackware managing some packages differently. 
We are going to fix them by applying a custom-made patch. It makes some slight changes in the source files to fix 
cmake linking errors and make sure qt5 is used.

Navigate to the `ROS-Slackware` repo you cloned previously. From there execute the script which will apply the patch. 
Note that the script expects you to have your workspace in '~/ros_catkin_ws':

```
./patch.sh
```

Note you might run into some errors if you did not choose desktop-full installation. Just ignore them.

###5.3 Build the workspace
We are now ready to build our workspace

```
cd ~/ros_catkin_ws
./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release
```

Wait for ages until it is done. 

#### Desktop-full install

If you are going for the desktop-full install and you have a 64bit version of Slackware, you are going to run into a build 
error which will look something like this:

```
==> Processing catkin package: 'gazebo_plugins'
...
Traceback (most recent call last):
  File "/home/niko/ros_catkin_ws/src/gazebo_ros_pkgs/gazebo_plugins/cfg/GazeboRosCamera.cfg", line 6, in <module>
    from dynamic_reconfigure.msg import SensorLevels
ImportError: No module named msg
CMakeFiles/gazebo_plugins_gencfg.dir/build.make:87: recipe for target '/home/niko/ros_catkin_ws/devel_isolated/gazebo_plugins/include/gazebo_plugins/GazeboRosCameraConfig.h' failed
make[2]: *** [/home/niko/ros_catkin_ws/devel_isolated/gazebo_plugins/include/gazebo_plugins/GazeboRosCameraConfig.h] Error 1
CMakeFiles/Makefile2:1489: recipe for target 'CMakeFiles/gazebo_plugins_gencfg.dir/all' failed
make[1]: *** [CMakeFiles/gazebo_plugins_gencfg.dir/all] Error 2
...
<== Failed to process package 'gazebo_plugins': 
  Command '['/home/niko/ros_catkin_ws/install_isolated/env.sh', 'make', '-j8', '-l8']' returned non-zero exit status 2
```

This error cannot be prevented beforehand because of a bug in the `catkin` build system which is currently being fixed. 
In the meantime, fix the error by navigating to the `ROS-Slackware` repo you cloned beforehand and execute:

```
./fixmodules.sh
```
Note that the script expects you to have your workspace in '~/ros_catkin_ws'.

Now get back to the ROS workspace and continue:

```
cd ~/ros_catkin_ws
./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release
```

Everything should install successfully

###5.3 Finalize installation

```
source ~/ros_catkin_ws/install_isolated/setup.bash
```

You are recommended to place this command in your `~/.bashrc` to make the changes permanent:

```
echo "~/ros_catkin_ws/install_isolated/setup.bash" >> ~/.bashrc
```

###5.4 Fix a possible build bug

If you are on a Slackware 64bit system, you must fix a future runtime problem which is result of
a bug in the `catkin` build system (currently being fixed). In particular some packages with the 
same name but different modules were built in both `install_isolated/lib64/python2.7/site-packages` 
and `install_isolated/lib/python2.7/site-packages`. This is incorrect as it causes python errors 
later. 

Navigate to the `ROS-Slackware` repo you cloned previously and execute from there:

```
./fixmodules.sh
```

The script will simply move all modules in `install_isolated/lib64/python2.7/site-packages`

##6. Enjoy ROS


#Issues

Make sure to check `PossibleIssues.md`. If you don't think your issue is related to the contents of the file,
file an issue to this repo.
