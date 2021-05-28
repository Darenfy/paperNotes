#### Prepare Host
- 至少 1 T 的 SSD硬盘
  - 分离有无 ASan 的 AOSP 工程
  - 需要保存 log
- 建议使用 Ubuntu 18.04

#### 准备 android 环境
- AOSP
  - 下载 AOSP 源码
  - 编译指定版本的源码
  - 编译带有 ASan 的源码
  - 刷机
- 需要做的
  - 下载源码
  - 检查版本
  - 下载合适的二进制文件

- 构建之前修改 ``/path/to/aosp/build/core/main.mk`` 使得 fuzz 更方便
  - ro.adb.secure=0 节省下每次刷机后都要人工点按屏幕信任主机的操作
  - persist.sys.disable_rescue=1 使援救部分失效 提升fuzz 效率
```
# line 273
## before modifying
ifneq (,$(user_variant))
  # Target is secure in user builds.
  ADDITIONAL_DEFAULT_PROPERTIES += ro.secure=1
  ADDITIONAL_DEFAULT_PROPERTIES += security.perf_harden=1

  ifeq ($(user_variant),user)
    ADDITIONAL_DEFAULT_PROPERTIES += ro.adb.secure=1
  endif
## after modifying
ifneq (,$(user_variant))
  # Target is secure in user builds.
  ADDITIONAL_DEFAULT_PROPERTIES += ro.secure=1
  ADDITIONAL_DEFAULT_PROPERTIES += security.perf_harden=1

  ADDITIONAL_DEFAULT_PROPERTIES += ro.adb.secure=0
  ADDITIONAL_DEFAULT_PROPERTIES += persist.sys.disable_rescue=1

  #ifeq ($(user_variant),user)
  #  ADDITIONAL_DEFAULT_PROPERTIES += ro.adb.secure=1
  #endif
```
- 安装 Android SDk 后，构建软链接[个人建议使用build之后原始版本的fastboot和adb]
```
sudo ln -s /path/to/sdk/platform-tools/adb /usr/bin/fastboot
sudo ln -s /path/to/sdk/platform-tools/adb /usr/bin/adb
```
- 分别刷有无asan的镜像
```
############################# Flash factory image       #############################
# Before flashing the manually build image, 
# you should flash the mobile phone with the corresponding factory image.
# please refer to the offical website for flashing factory image.

############################# Flash AOSP image without ASan #############################

# we need to compile aosp in a bash environment
bash

cd /path/to/aosp
# prepare environment
source build/envsetup.sh
# select the target version.
# 50 corresponding to the aosp_taimen-userdebug
# you can use lunch to see the allowed choices.
lunch 50

# compile AOSP and save the compile commands
# replace the N_PROCS with the number you want, 
# e.g., make -j15 showcommands 2>&1 >cmd.txt
make -j [N_PROCS] showcommands 2>&1 >cmd.txt

## here, you should run your commands to flash the image.

############################# Flash AOSP image with ASan #############################

cd ..
# copy the entire project to another place.
cp /path/to/aosp /path/to/aosp_asan
cd /path/to/aosp_asan
source build/envsetup.sh
lunch 50

# compile the entire AOSP with ASan enabled
# replace the N_PROCS with the number you want, 
# e.g., SANITIZE_TARGET=address make -j15
SANITIZE_TARGET=address make -j [N_PROCS]

## here, you should run your commands to flash the image with ASan enabled.
```
#### 配置 FANS
使用模版配置 FANS，需要配置如下选项
- fans_dir -> FANS 目录
- aosp_dir -> AOSP 目录
- aosp_sanitizer_dir -> 带有 ASan 的 AOSP 目录
- aosp_compilation_cmd_file -> AOSP 编译命令文件路径
- lunch_command -> 构建数
- aosp_clang_location -> 相对于 aosp_dir 用来编译 AOSP 的 clang 工具路径，例如 ``prebuilts/clang/host/linux-x86/clang-4691093/bin/clang++.real``
- manually_build_clang_location -> 手动构建 clang 的路径 
- clang_plugin_option -> 编译命令的额外选项 去加载 clang 插件
- service_related_file_collector_workdir -> 服务相关文件收集器的工作路径 默认
- service_related_filepath_storage_location -> 保存服务相关文件 默认
- misc_parcel_related_function_storage_location -> 保存拥有 parcel 参数的混合函数 默认
-  special_parcelable_function_storage_location -> 保存特殊包结构的特殊函数 默认
-  aosp_compilation_cc1_cmd_file -> 保存 cc1 命令 默认
-  already_preprocessed_files_storage_location -> 保存已经经过预处理的文件 默认
-  rough_interface_related_data_dir -> 保存预处理过程中抽取的数据，位于 aosp 根目录，名称为data
-  already_parsed_interfaces_storage_location -> 保存后处理过程中已经解析过的接口，默认
-  interface_model_extractor_tmp_dir -> 接口模型抽取器使用的 tmp 目录，默认
-  interface_model_extractor_dir -> 接口模型抽取器的工作目录，默认
-  interface_dependency_dir -> 接口依赖关系目录，默认

#### 服务相关文件收集器
这一步是收集 android 本地系统服务相关的文件
- 顶层接口和多层接口
- 标准的解析和平坦化结构
- 拥有包参数的混合函数
- 不使用 readFromParcel 或者 writeToParcel 方法的特殊打包结构
使用如下命令直接收集
```
python collector.py
```
<!--
排除了由 AIDL 工具集为 32 位程序生成的文件
-->
一开始，我们并不知道混合函数和特定打包结构的信息。
由于一个混合函数有可能调用其他的混合函数，所以我们应该递归收集
- 运行 collector.py 脚本
- 使用接口模型抽取器抽取接口模型。此过程中，接口模型抽取器将会记录混合函数和特殊打包结构
- 如果没有发现新函数，则停止；否则重新运行 collector.py 脚本
<!--
从另一个角度来看，尽管 AOSP 是由 Google 领导，但还是有很多非标准代码，否则只需要运行 collecotr.py 脚本就可以收集所有的服务相关文件
-->
collector.py 运行流程见 ./collector.py 

#### 接口模型抽取
这一部分主要是抽取函数模型，分为预处理和后处理
在进行抽取前，需要创建``workdir/interface-model-extractor``文件夹
#####    预处理
预处理解决的是基于ast将接口相关信息抽取出来并转换成适当的格式
* python gen_all_related_cc1_cmd.py
* 下载 android llvm —> 特定版本
* 编译 Clang 插件 BinderIface
    * 在已下载llvm的文件夹中创建BinderIface的软连接
    * 在CMakeLists.txt 文件中添加语句 add_subdirectory(BinderIface)
    * 编译BinderIface插件
    * <!--可能在编译BinderIface之前需要从源码中构建Z3-->
* 处理边角类型：有很少一部分特殊的例子不容易处理，需要书写大量的代码，所以这里在不影响语义的情况下手动修改；另外由于一个包装结构的 readFromParcel 函数和 writeToParcel 函数的变量类型相匹配，而非标准代码的存在影响了发现变量类型的准确性，利用这两个函数处理变量的顺序是一致的这一特性尽可能使得变量更精准。
* 基于抽象语法树抽取接口相关信息
    * 创建符号链接
    * 使用 extract_from_ast.py 抽取相关信息：建议第一次使用时选择 y，以后当要处理新文件时选择 n 以节省时间
#####    后处理
后处理将会生成精确的接口模型
* 安装所需要的python库
* 手动修改xmljson
* 运行 sh postprocess.sh 产生接口模型：当发现新函数时，说明应该收集在服务相关文件收集器中的描述的函数相关文件
####    依赖关系推断器
将推断出事务间的变量依赖关系和接口依赖关系
* 安装 graphviz 包依赖
* 运行 infer_dependency.sh 推断依赖关系并生成接口依赖关系图表
####    fuzzer 引擎
引擎包含两部分
* fuzzer：用来fuzz安卓原生系统服务的真正fuzzer
* manager：在某些方面管理fuzz的自动运行
    * 将fuzz和数据push到手机中
    * 从手机中同步崩溃日志
    * 刷机
#####    准备数据
* 接口模型
* 不同的种子集
    * 文件
        * media files
        * apk files
        * misc files
    * media URLs
    * package name list
    * permission list
#####    构建 fuzzer
* 根据目标机型修改 android.bp 中的 include_dirs
* 根据 setup.template.sh 构建fuzzer
#####    测试 fuzzer
根据 test_fuzzer.template.sh 配置适应环境进行测试
#####    fuzzer manager
已经实现自动重新刷机，若要使用需要单独构建，提供相应镜像，并对应调整 fuzzer manager
* 安装所需库
* 准备手机
* 刷机：利用 flash.py 刷机前需要配置 flash.cfg 
* 配置 fuzzer manager —> fuzzer.cfg
* 运行 manager.py
* fuzz 结果存于 root_log_dir 中
####    结果
workdir 目录包含以下结果
* 服务相关文件信息
* 接口模型
* 简化的接口依赖关系