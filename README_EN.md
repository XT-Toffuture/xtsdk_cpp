# xtsdk_cpp V2.X
    Adapted for lidar Firmware Version 2.x, SDK updated to version 2.x.x, compatible with 1.x firmware version.

## News
2025.01.06:
Updated multi-frequency feature, allowing adjustment of corresponding frequency settings for three integration time tiers: bool setMultiModFreq(ModulationFreq freqType1, ModulationFreq freqType2, ModulationFreq freqType3, ModulationFreq freqType4 = FREQ_24M);
For firmware version 2.20 and above, can read key parameters stored internally in the lidar: bool getDevConfig(RespDevConfig &devConfig);
Added ability to receive grayscale images in non-grayscale mode through API: bool setAdditionalGray(const uint8_t &on);
Introduced dust filtering optimization for moving objects and near-range point cloud optimization: void setPostProcess(const float &dilation_pixels, const uint8_t &mode, const double &mode_th);
# SDK Description
This code is confidential.

## Introduction
    XTSDK provides an easy-to-use API for developers to quickly integrate and develop applications by encapsulating the communication process with the lidar (communication, packetization, depth-to-point cloud conversion).

## File List
    xtsdk.h SDK user interface API
    frame.h Frame structure definition, including pixel data, point cloud data, image size, time, temperature, type, etc.
    xtdaemon.h SDK multithreading
    communication.h Communication interface definitions
    communicationNet.* Ethernet communication implementation
    communicationUsb.* USB communication implementation
    utils.* Utility classes
    cartesianTransform.* Depth-to-point cloud implementation
    baseFilter.* Basic filtering implementation
## Compilation Environment
### Windows:
    Language: C++ 17 or above
    Dependency Library: boost_1_78_0 or above
    Compilation Configuration: cmake 3.21 and above
    Compiler: msvc 2019, mingw 8.10
    
    The following third-party dependency libraries and compilation tools are required in the msvc2019 Windows environment:
    - cmake: cmake-3.23.3-windows-i386.zip
      - Download link: https://github.com/Kitware/CMake/releases/download/v3.23.3/cmake-3.23.3-windows-i386.zip
    - Compiler:
      - Download Build Tools for Visual Studio 2019 (version 16.0) x64 version from the Visual Studio website.
      - After downloading, install with default options.
      
    - Boost Library: boost_1_79_0-msvc-14.2-64.exe
      - Download link: https://boostorg.jfrog.io/artifactory/main/release/1.79.0/binaries/boost_1_79_0-msvc-14.2-64.exe
      
    - OpenCV Library: opencv-4.6.0-vc14_vc15.exe
      - Download link: https://opencv.org/releases/
      - https://sourceforge.net/projects/opencvlibrary/files/4.6.0/opencv-4.6.0-vc14_vc15.exe/download
      
    - PCL Library: PCL-1.12.1-AllInOne-msvc2019-win64.exe
      - Download link: https://github.com/PointCloudLibrary/pcl/releases/download/pcl-1.12.1-rc1/PCL-1.12.1-rc1-AllInOne-msvc2019-win64.exe
    - Compilation: cmake-gui.exe
      - You can use cmake-gui.exe to specify the xtsdk_cpp directory, then specify the output directory, and select the compiler to compile.
### Linux:
    x86: Ubuntu 18.04 and above         
    aarch64: Ubuntu 20.04 and above (aarch64 below 20.04 does not support filtering algorithms)
    - Boost
        sudo apt install libboost-all-dev
    
    - PCL
        If point cloud display is not required, it can be omitted.  
        sudo apt install libpcl-dev
    - OpenCV
        Point cloud transformation uses cv to process intrinsic parameters, filtering uses cv's median filter: sudo apt install libopencv-dev
## Compilation Method
### Linux:
    cd ~/.xtsdk_cpp
    mkdir build && cd build
    cmake ..
    
    make
    The example is generated by default in ~/.xtsdk_bin_release.
    
### Windows:
    cd C:\Users\YourUsername.xtsdk_cpp
    mkdir build && cd build
    cmake .. -G "Visual Studio 16 2019"  # Visual Studio 2019 version
    
    make
    The example is generated by default in C:\Users\YourUsername\xtsdk_bin_release.

## SDK Interface API Description
### Status Codes:
    SDK Status
    STATE_UNSTARTUP     SDK not started
    STATE_PORTOPENING   Attempting to open communication port
    STATE_TXRX_VERIFYING Communication port opened, verifying communication
    STATE_UDPIMGOK      Received UDP data, but TCP is attempting to connect
    STATE_CONNECTED     Device connected (communication normal)
    STATE_UNKNOW        Unknown state, theoretically should not occur
### Lidar Device Status
    Refer to DevStateCode definition in xtsdk.h.
    Command Return Status
    Refer to CmdRespCode definition in xtsdk.h.
    Event Data Types:
    sdkState    SDK status has changed
    devState    Device status has changed
### Callback Functions
    void setCallback(std::function<void(const std::shared_ptr<CBEventData> &) >  eventcallback, std::function<void(const std::shared_ptr<Frame> & ) >  imgcallback);

### Sets two callback functions for receiving SDK events and device reporting information:
    eventcallback: User receives SDK events, device reported information, logs, etc.
    imgcallback: User receives depth, grayscale, and point cloud frames from the device.
## Xintan LiDAR SDK Data Interpretation
    The Xintan LiDAR is a Flash type, where the entire sensor has a global exposure, and all points are captured at the same time without the concept of Ring channels.
    We output the point cloud format as XYZI, with a timestamp for the entire frame.
    
    The point cloud coordinate system is the same as the Xintan lidar coordinate system, with the lidar at the origin.
    
    Camera data is passed in the frame object received in imgCallback:
    frame data description:
    
    uint16_t width; // Data resolution width
    uint16_t height; // Data resolution height
    uint64_t timeStampS; // Frame timestamp: seconds
    uint32_t timeStampNS; // Frame timestamp: nanoseconds
    uint16_t temperature; // Sensor temperature
    uint16_t vcseltemperature; // Light board temperature
    std::vector<uint16_t> distData; // Sorted distance data, depth map arranged from top-left to bottom-right for all pixels; valid values are 0~964000, values above 964000 indicate the pixel is invalid.
    std::vector<uint16_t> amplData; // Sorted signal strength data, signal strength arranged from top-left to bottom-right for all pixels; valid values are 0~2039, values above 64000 indicate the pixel is invalid.
    std::vector<XtPointXYZI> points; // Point cloud data, unordered point cloud, arranged similarly to distData and amplData according to pixels from top-left to bottom-right.
    The point data format in points is XYZI, for example, the z-direction value of the point at row 10, column 20 is points[width*10 + 20].z, undistorted.

    ***NOTE***
    std::vector<uint16_t> distData is the filtered depth array; rawdistData is the depth array before filtering. The above depth data is not undistorted.
## C++ Example Overview
    The Xintan SDK provides two example programs' source codes to facilitate quick development of application software:
    
    - sdk_example:
    Implements the most basic source code to connect to the device for measurement, only displaying logs on receipt of images, without drawing.
    
    - sdk_example_pcl:
    Implements data retrieval from the device and visualizes the point cloud using PCL.
    
    - sdk_example_play:
    Reads xbin files and displays point clouds.
    
    - sdk_example_record:
    Records xbin files.
    
    - sdk_example_ufw:
    Online firmware upgrade.
    
### NOTE
    
    sdk_example_pcl and sdk_example_play depend on PCL and VTK for rendering. If a core dump occurs in ARM or some x86 environments after running:
    Please check if the machine supports hardware GPU rendering or check the compatibility of OpenGL with VTK and PCL libraries. It is recommended to use stable versions of PCL 1.14 and VTK 9.1 for rendering in ARM.
    If the above does not resolve the issue, input export LIBGL_ALWAYS_SOFTWARE=1 in the terminal before running, which may increase CPU usage and affect point cloud frame rates, but not data frame reduction.
    NOTE
    
    - For sdk_example_pcl:
    Run Parameters 1: lidar address
    Run Parameters 2: 1 to use local configuration file xtsdk_cpp/sdk_example/cfg/xintan.xtcfg ignoring lidarsaved parameters; 0 to use parameters saved on the lidar, ignoring the local configuration file xtsdk_cpp/sdk_example/cfg/xintan.xtcfg.
    For example: ./sdk_example_pcl 192.168.0.101 0
    
    - For sdk_example_play:
    Copy the xbin recording folder a to the same directory as the executable.
    For example: ./sdk_example_play a/
    
    - For sdk_example_record:
    Run Parameters 1: lidar address
    Run Parameters 2: Path to save the xbin files
    Run Parameters 3: 1 to use the local configuration file xtsdk_cpp/sdk_example/cfg/xintan.xtcfg ignoring lidar saved parameters; 0 to use parameters saved on the lidar, ignoring the local configuration file xtsdk_cpp/sdk_example/cfg/xintan.xtcfg.
    For example: ./sdk_example_record 192.168.0.101 ./ 0
    
    - For sdk_example_ufw:
    Copy firmware *.bin files to the same directory as the executable.
    Run Parameters 1: lidar address
    Run Parameters 2: Path to *.bin
    For example: ./sdk_example_ufw 192.168.0.101 *.bin
    
    Manually restart the device.
    
    In a Linux environment, if using serial communication, the lidar address should be /dev/ttyACM+number, usually /dev/ttyACM0.
    Steps are as follows:
    1. ls /dev/tty* to find /dev/ttyACM+number. If it does not exist, it indicates that the lidar is not properly connected via serial in Linux. Please check the connection.
    
    2. sudo chmod 777 /dev/ttyACM+number (usually /dev/ttyACM0)
    3. ./sdk_example_pcl /dev/ttyACM+number 0
    For upgrading firmware, use sudo ./sdk_example_ufw /dev/ttyACM+number *.bin.

## SDK Running API
- bool setConnectIpaddress(std::string ipAddress);

If connecting via network, use this API to set the IP address of the device to connect to (e.g., "192.168.0.101").

- bool setConnectSerialportName(std::string serialportName);

If connecting via USB, use this API to set the COM port address of the device to connect to (e.g., "COM2").

- void startup();

Starts the SDK.

- void shutdown();

Disconnects the device.

- bool isconnect();

Checks if the device connection is successful.

- bool isUsedNet();

Checks if the device is connected over the network.

- SdkState getSdkState();

Retrieves the SDK state.

- SdkState getDevState();

Retrieves the device state.

- std::string getStateStr();

Retrieves the SDK state as a string.

- int getfps();

Retrieves the calculated frame rate of the SDK.

- bool setSdkReflectiveFilter(const float &threshold_min, const float &threshold_max);
Sets the Reflective Filter in the SDK.

- bool setSdkKalmanFilter(uint16_t factor, uint16_t threshold);

Sets the Kalman filter in the SDK.

- bool setSdkEdgeFilter(uint16_t threshold);

Sets the outlier filter in the SDK.

- bool setSdkMedianFilter(uint16_t size);

Sets the median filter in the SDK.

- void setPostProcess(const float &dilation_pixels, const uint8_t &mode);

Sets near-range point cloud optimization and moving object optimization.

- bool clearAllSdkFilter();

Clears all filter settings in the SDK.

- bool setSdkCloudCoordType(ClOUDCOORD_TYPE type);

Sets the coordinate system for outputting point clouds in the SDK.

### Command Related APIs
- bool testDev();

​  Tests device command interaction.

- bool start(ImageType imgType, bool isOnce = false);

​  Specifies the expected image type for measurement; isOnce indicates whether to acquire data once.

- bool stop();

​  Stops the device measurement.

- bool getDevInfo(RespDevInfo & devInfo);

​  Retrieves device information; see the data structure definition in xtsdk.h.

- bool getDevConfig(RespDevConfig & devConfig);

​  Retrieves device configuration information; see the data structure definition in xtsdk.h.

- bool setIp(std::string ip, std::string mask, std::string gate);

​  Sets the device's IP address configuration:

​  ip such as "192.168.0.101",

​  mask such as "255.255.255.0",

​  gate such as "192.168.0.1".

- bool setFilter(uint16_t temporal_factor, uint16_t temporal_threshold, uint16_t edgefilter_threshold);

​  Sets the basic filtering parameters in the device:

​  temporal_factor Kalman factor, maximum 1000, typically set to 300;

​  temporal_threshold Kalman threshold, maximum 2000, typically set to 300;

​  edgefilter_threshold Outlier threshold, maximum 2000, typically set to 0.

- bool setIntTimesus(uint16_t timeGs, uint16_t time1, uint16_t time2, uint16_t time3);

Sets integration time in microseconds:

​  timeGs integration time when measuring grayscale;

​  time1 first integration time;

​  time2 second integration time (valid only when using HDR, set to 0 to disable);

​  time3 third integration time (valid only when HDR is set to HDR_TAMPORAL mode, set to 0 to disable).

- bool setMinAmplitude(uint16_t minAmplitude);

​  Sets the lower limit for valid signal amplitude, typically set between 50 and 100.

- bool setHdrMode(HDRMode mode);

​  Sets HDR type; mode can be configured as:

​  HDR_OFF to disable HDR;

​  HDR_TAMPORAL for temporal mode.

- bool resetDev();

​  Restarts the device.

- bool setModFreq(ModulationFreq freqType);

​  Sets the device modulation frequency:

​  freqType: FREQ_12M, FREQ_6M.

- bool setRoi(uint16_t x0, uint16_t y0, uint16_t x1, uint16_t y1);

​  Sets the ROI region.

- bool setUdpDestIp(std::string ip, uint16_t port = 7687);

Sets the UDP destination IP address.

- bool setMaxFps(uint8_t maxfps);

Sets the maximum measurement frame rate of the device (the actual frame rate in HDR mode should be a multiple of the desired frame rate).

- bool getLensCalidata(CamParameterS & camparameter);

Retrieves lens intrinsic parameters.

- bool getDevConfig(RespDevConfig &devConfig);

Retrieves device configuration information.

- bool setAdditionalGray(const uint8_t &on);

Sets whether to obtain grayscale images.

- bool setMultiModFreq(ModulationFreq freqType1, ModulationFreq freqType2, ModulationFreq freqType3, ModulationFreq freqType4 = FREQ_24M);

Sets the corresponding frequencies for three integration time tiers.



- bool setBinningV(uint8_t flag);

Sets the vertical binning mode.
    0: Do not set binningV.
    1: Set binningV.

- bool getImuExtParamters(ExtrinsicIMULidar &imuparameters, uint8_t flag = 1);

Retrieves the extrinsic parameters from IMU to LiDAR.
    0: Get the default extrinsic parameters.
    1: Get the calibrated extrinsic parameters.
    If you need to call setTransMirr, make sure to set it first before retrieving the extrinsic parameters.
四
