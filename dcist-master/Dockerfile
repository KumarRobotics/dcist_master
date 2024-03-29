# The upstream image depends on the architecture, and it is defined when
# building the image. Please see build.bash for more details
ARG UPSTREAM_IMAGE
FROM $UPSTREAM_IMAGE
ARG ARCHITECTURE
ARG BARE_OR_NVDA

#Run the frontend first so it doesn't throw an error later
RUN apt-get update \
 && export TZ="America/New_York" \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y keyboard-configuration \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y tzdata \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y locales \
 && ln -fs "/usr/share/zoneinfo/$TZ" /etc/localtime \
 && dpkg-reconfigure --frontend noninteractive tzdata \
 && apt-get clean

# General dependencies for development
RUN apt-get update \
 && apt-get install -y \
        build-essential \
        cmake \
        cppcheck \
        gdb \
        git \
        libeigen3-dev \
        g++ \
        libbluetooth-dev \
        libcwiid-dev \
        libgoogle-glog-dev \
        libspnav-dev \
        libusb-dev \
        lsb-release \
        mercurial \
        python3-dbg \
        python3-empy \
        python3-pip \
        python3-venv \
        software-properties-common \
        sudo \
        wget \
	      curl \
        cmake-curses-gui \
        geany \
        tmux \
        dbus-x11 \
        iputils-ping \
        default-jre \
        iproute2 \
 && apt-get clean

# Install a good neovim version
RUN add-apt-repository -y ppa:neovim-ppa/stable \
        && apt-get update \
        && DEBIAN_FRONTEND=noninteractive apt-get install -y neovim

# Add a user with the same user_id as the user outside the container
# Requires a docker build argument `user_id`
ARG user_id=1000
env USER dcist
RUN useradd -U --uid ${user_id} -ms /bin/bash $USER \
 && echo "$USER:$USER" | chpasswd \
 && adduser $USER sudo \
 && echo "$USER ALL=NOPASSWD: ALL" >> /etc/sudoers.d/$USER

# Set locales
RUN sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen && \
    locale-gen
ENV LC_ALL en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en

# Commands below run as the developer user
USER $USER

# Remove all opencv packages installed by default
RUN sudo apt-get purge -y '*opencv*'

# Install ROS Noetic
RUN sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list' \ && sudo /bin/sh -c 'curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | apt-key add -' \
 && sudo apt-get update \
 && sudo apt-get install -y \
    python3-catkin-tools \
    python3-rosdep \
    python3-rosinstall \
    ros-noetic-desktop-full

RUN sudo rosdep init \
 && sudo apt-get clean

RUN rosdep update

# Install ROS dependencies
RUN sudo apt-get install -y \
    ros-noetic-nmea-msgs \
    ros-noetic-nmea-navsat-driver \
    ros-noetic-rosserial \
    ros-noetic-rosserial-server \
    ros-noetic-teleop-twist-joy \
    ros-noetic-lms1xx \
    ros-noetic-costmap-2d \
    ros-noetic-gtsam \
    ros-noetic-octomap \
    ros-noetic-pcl-ros \
    ros-noetic-octomap-msgs \
    ros-noetic-vision-msgs \
    ros-noetic-mavros-msgs \
    ros-noetic-mavros

# Install geographiclib_datasets required by mavros
RUN cd /opt/ros/noetic/lib/mavros/ && sudo ./install_geographiclib_datasets.sh

# Install pytorch for the appropriate architecture
RUN if [ "$BARE_OR_NVDA" = "nvda" ] ; then \
  if [ "$ARCHITECTURE" = "linux/amd64" ] ; then \
    echo "nvda - x86_64"; sudo pip3 install torch==1.8.2 torchvision==0.9.2 torchaudio==0.8.2 --extra-index-url https://download.pytorch.org/whl/lts/1.8/cu111 ; \
  else \
    echo "nvda - arm64" ; echo "Skipping pytorch installation for Jetson"; \
  fi; \
else \
  if [ "$ARCHITECTURE" = "linux/amd64" ] ; then \
    echo "bare - x86_64"; sudo pip3 install torch==1.8.2 torchvision==0.9.2 torchaudio==0.8.2 --extra-index-url https://download.pytorch.org/whl/lts/1.8/cpu ; \
  else \
    echo "Architecture not supported"; exit 1; \
  fi; \
fi;

# Install dependencies for MOCHA
RUN sudo sh -c 'pip3 install zmq rospkg defusedxml utm'

# Install all dependencies for ROFL. Based on the ROFL build action
# https://github.com/versatran01/rofl-beta/blob/main/.github/workflows/build.yaml
RUN mkdir -p ~/cpp_deps
RUN cd ~/cpp_deps \
  && git clone --depth 1 --branch v0.6.0 https://github.com/google/glog.git \
  && cd glog \
  && cmake -S . -B build -G "Unix Makefiles" -DCMAKE_CXX_STANDARD=17 \
  && cmake --build build \
  && sudo cmake --build build --target install
RUN cd ~/cpp_deps \
  && git clone --depth 1 --branch 8.1.0 https://github.com/fmtlib/fmt.git \
  && cd fmt && cmake -DCMAKE_POSITION_INDEPENDENT_CODE=TRUE -DCMAKE_CXX_STANDARD=17 -DFMT_TEST=False \
  && sudo make install
RUN cd ~/cpp_deps \
  && git clone --depth 1 --branch 20220623.0 https://github.com/abseil/abseil-cpp.git \
  && cd abseil-cpp \
  && mkdir build \
  && cd build \
  && cmake -DABSL_BUILD_TESTING=OFF -DCMAKE_CXX_STANDARD=17 -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_POSITION_INDEPENDENT_CODE=ON .. \
  && sudo sudo cmake --build . --target install
RUN cd ~/cpp_deps \
  && git clone https://github.com/strasdat/Sophus.git \
  && cd Sophus \
  && git checkout 785fef3 \
  && mkdir build \
  && cd build \
  && cmake -DBUILD_SOPHUS_TESTS=OFF -DBUILD_SOPHUS_EXAMPLES=OFF -DCMAKE_CXX_STANDARD=17 .. \
  && sudo make install -j8
RUN cd ~/cpp_deps \
  && git clone https://github.com/google/benchmark.git \
  && cd benchmark && cmake -E make_directory "build" \
  && cmake -E chdir "build" cmake -DBENCHMARK_DOWNLOAD_DEPENDENCIES=on -DCMAKE_BUILD_TYPE=Release ../ \
  && sudo cmake --build "build" --config Release --target install
RUN cd ~/cpp_deps \
  && git clone --recursive https://github.com/stevenlovegrove/Pangolin.git \
  && cd Pangolin \
  && git checkout v0.9.1 \
  && sudo apt install -y $(./scripts/install_prerequisites.sh required --list) \
  && cmake -B build \
  && sudo cmake --build build --config Release --target install

# Build Ian's ORBSLAM3 version with a few patches (thanks Varun)
RUN mkdir ~/Repos && cd ~/Repos \
 && git clone https://github.com/iandouglas96/ORB_SLAM3.git && cd ORB_SLAM3 \
 && rm -rf Thirdparty/Sophus \
 && sed -i '/${PROJECT_SOURCE_DIR}\/Thirdparty\/Sophus/c\#${PROJECT_SOURCE_DIR}\/Thirdparty\/Sophus' CMakeLists.txt \
 && sed -i '/#include "Thirdparty\/Sophus\/sophus\/geometry.hpp"/c\#include "sophus\/geometry.hpp"' include/Frame.h \
 && sed -i '/#include "Thirdparty\/Sophus\/sophus\/geometry.hpp"/c\#include "sophus\/geometry.hpp"' include/Converter.h \
 && sed -i '/#include "Thirdparty\/Sophus\/sophus\/sim3.hpp"/c\#include "sophus\/sim3.hpp"' include/Converter.h \
 && sed -i '/find_package(OpenCV 3)/c\find_package(OpenCV 4)' CMakeLists.txt \
 && sed -i '/CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)/c\CHECK_CXX_COMPILER_FLAG("-std=c++14" COMPILER_SUPPORTS_CXX14)' CMakeLists.txt \
 && sed -i '/if(COMPILER_SUPPORTS_CXX11)/c\if(COMPILER_SUPPORTS_CXX14)' CMakeLists.txt \
 && sed -i '/set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")/c\set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")' CMakeLists.txt \
 && sed -i '23 i #include<iostream>' include/ImuTypes.h \
 && sed -i '23 i #include<iostream>' include/GeometricTools.h \
 && sed -i '126 i -lfmt' CMakeLists.txt \
 && chmod +x build.sh && ./build.sh
RUN cd ~/Repos/ORB_SLAM3/Vocabulary/ && tar xf ORBvoc.txt.tar.gz

# Clone all the gazebo models so that you don't download them every time
# docker is run
RUN mkdir -p /home/$USER/.gazebo && git clone https://github.com/osrf/gazebo_models /home/$USER/.gazebo/models

# When running a container start in the developer's home folder
WORKDIR /home/$USER

# Configure .bashrc
RUN echo 'export PS1="\[$(tput setaf 2; tput bold)\]\u\[$(tput setaf 7)\]@\[$(tput setaf 3)\]\h\[$(tput setaf 7)\]:\[$(tput setaf 4)\]\W\[$(tput setaf 7)\]$ \[$(tput sgr0)\]"' >> ~/.bashrc
RUN echo 'source ~/.bashrc_host' >> ~/.bashrc

# Fix permissions of mounted folders
RUN /bin/sh -c 'echo sudo chown $USER:$USER ~/user_ws >> ~/.bashrc'
RUN /bin/sh -c 'echo sudo chown $USER:$USER ~/data >> ~/.bashrc'
RUN /bin/sh -c 'echo sudo chown $USER:$USER ~/.ros >> ~/.bashrc'

# Source ROS repositories
RUN /bin/sh -c 'echo ". /opt/ros/noetic/setup.bash" >> ~/.bashrc' \
 && /bin/sh -c 'echo ". /usr/share/gazebo/setup.sh" >> ~/.bashrc' \
 && /bin/sh -c 'echo "source ~/user_ws/devel/setup.bash" >> ~/.bashrc'

# Copy the dcist_ws environment and build it
COPY --chown=$USER:$USER ../ws /home/$USER/dcist_ws
RUN cd ~/dcist_ws \
 && catkin config --extend /opt/ros/noetic \
 && catkin build --no-status -DCMAKE_BUILD_TYPE=Release
