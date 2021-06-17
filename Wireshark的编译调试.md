## Wireshark的编译调试

### Linux平台

主要是针对Ubuntu，如果是其它的Linux平台，要求不太一样

#### 环境要求

```
sudo bash tools/debian-setup.sh --install-optional
sudo bash tools/debian-setup.sh --install-deb-deps
sudo bash tools/debian-setup.sh --install-test-deps
```

修改顶层CMakeLists.txt，在文件开始增加，修改CMake为调试模式

```
SET(CMAKE_BUILD_TYPE "Debug")
SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g -ggdb")
SET(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -O3 -Wall")
```

安装json解析包

```
sudo apt-get install libjson-glib-1.0-0 libjson-glib-dev pkg-config
```

#### 修改CMakeLists.txt

为了适配新增的函数文件，需要在CMakeLists.txt，在BUILD_tshark中增加我们需要编译的文件，并且增加对json_glib的要求

```
if(BUILD_tshark)
	set(tshark_LIBS
		ui
		capchild
		caputils
		wiretap
		epan
		${VERSION_INFO_LIBS}
		${APPLE_CORE_FOUNDATION_LIBRARY}
		${APPLE_SYSTEM_CONFIGURATION_LIBRARY}
		${WIN_WS2_32_LIBRARY}
		${M_LIBRARIES}
	)
	set(tshark_FILES
		$<TARGET_OBJECTS:capture_opts>
		$<TARGET_OBJECTS:cli_main>
		$<TARGET_OBJECTS:shark_common>
		$<TARGET_OBJECTS:version_info>
		tshark-tap-register.c
		tshark.c
		#new proto
		newproto/jsonopt.c
		newproto/register_protocol.c
		${TSHARK_TAP_SRC}
	)
	#find json-glib
	set(ENV{PKG_CONFIG_PATH} /usr/lib/x86_64-linux-gnu/pkgconfig)
	find_package(PkgConfig REQUIRED)
	pkg_search_module(JSON REQUIRED json-glib-1.0)
	include_directories(${JSON_INCLUDE_DIRS})
	link_directories(${JSON_LIBRARY_DIRS})
	
	set_executable_resources(tshark "TShark" UNIQUE_RC)
	add_executable(tshark ${tshark_FILES})
	set_extra_executable_properties(tshark "Executables")
	#add lib flag
	target_link_libraries(tshark ${tshark_LIBS} ${JSON_LIBRARY_DIRS} ${JSON_LDFLAGS})
	install(TARGETS tshark RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()
```

#### 编译运行

```
mkdir Development
cd Development
cmake -DCMAKE_BUILD_TYPE=Debug ..
make
make install
```

最终会在run文件夹中生成可执行wireshark

### Windows平台

基本上是按照Wireshark官网的安装路径来做的

#### 环境要求

1、安装Chocolatey，这是一个Windows平台的包管理程序

2、安装Visual Studio 2019或2017
3、安装QT，我用的是5.9.5的开源版，放在192.168.2.195的共享文件夹上，可以直接下载

4、安装Python3，可以用choco安装

```
choco install -y python3
```

5、安装Perl

```
choco install -y strawberryperl
```

6、安装Git，可以用choco安装，也可自行安装

```
choco install -y git
```

7、安装Cmake

```
choco install -y cmake
```

8、安装一系列程序

```
choco install -y asciidoctorj xsltproc docbook-bundle
choco install -y winflexbison3
```

#### 下载代码

```
cd C:\Development
git clone https://gitlab.com/wireshark/wireshark.git
```

#### 设置环境变量

打开Visual Stiudio提供的64位命令行工具，设置环境变量，环境变量中的地址需要适配于自己主机相应软件的位置

```
set WIRESHARK_BASE_DIR=C:\Development
rem set WIRESHARK_LIB_DIR=c:\Development\wireshark-win64-libs
set QT5_BASE_DIR=C:\Qt\Qt5.9.5\5.9.5\msvc2017_64
mkdir C:\Development\wsbuild64
cd C:\Development\wsbuild64
```

#### 进行Cmake编译

```
cmake -G "Visual Studio 16 2019" -A x64 ..\wireshark
```

编译成功后打印出

```
--Configuring done
-- Generating done
-- Build files have been written to: C:/Development/wsbuild64
```

Build成VS项目

```
msbuild /m /p:Configuration=RelWithDebInfo Wireshark.sln
```

### 问题

Windows环境编译成功后生成了exe文件，可以正常运行，但是还没有成功在VS下进行调试工作

Linux环境成功编译运行，可以通过gdb调试