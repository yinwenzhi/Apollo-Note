# 前言
Apollo各个模块之间节点通信的一种方式是采用ROS节点和topic的方式，
通过adapter 来管理节点和topic的订阅关系。
通过adaptermanager来管理adapter。
# 处理流程
Apollo适配器需要在对应的模块中初始化，举例在Planning模块中，AdapaterManager的初始化需要拿Planning模块的适配器配置文件，去实例化Adapter。

1. main.cc中
```
// 代码 modules\planning\main.cc 中

APOLLO_MAIN(apollo::planning::Planning)
```
这是一个宏定义
```
#define APOLLO_MAIN(APP)                                     \
  int main(int argc, char **argv) {                          \
    google::InitGoogleLogging(argv[0]);                      \
    google::ParseCommandLineFlags(&argc, &argv, true);       \
    signal(SIGINT, apollo::common::apollo_app_sigint_handler)\
    APP apollo_app_;                                         \
    ros::init(argc, argv, apollo_app_.Name());               \
    apollo_app_.Spin();                                      \
    return 0;                                                \
  }

#endif  // MODULES_APOLLO_APP_H_
```
从中间可以看到包括几个步骤
- 始化Google日志工具
- 使用Google命令行解析工具解析相关参数
- 注册接收中止信号“SIGINT”的处理函数：apollo::common::apollo_app_sigint_handler
（该函数的功能十分简单，就是收到中止信号“SIGINT”后，调用ros::shutdown()关闭ROS）
- 创建apollo::planning::Planning对象：apollo_app_  
(实例化决策模块，初始化ROS决策节点)
- 初始化ROS环境
- 调用(基础模块类)apollo_app_.Spin()函数开始消息处理循环



2. apollo_app_.Spin()函数开始消息处理循环
这是Apollo应用的入口，完成初始化、启动、关闭（当ros要求关闭时）应用
所有模块的类，均基于Apollo_App基础模块类
```
modules/common/apollo_app.cc 基类回旋方法

int ApolloApp::Spin() {
  ros::AsyncSpinner spinner(1); // 初始化1个线程
  auto status = Init(); 
  if (!status.ok()) {
    AERROR << Name() << " Init failed: " << status;
    ReportModuleStatus(apollo::hmi::ModuleStatus::UNINITIALIZED);
    return -1;
  }
  ReportModuleStatus(apollo::hmi::ModuleStatus::INITIALIZED);
  status = Start();
  if (!status.ok()) {
    AERROR << Name() << " Start failed: " << status;
    ReportModuleStatus(apollo::hmi::ModuleStatus::STOPPED);
    return -2;
  }
  ReportModuleStatus(apollo::hmi::ModuleStatus::STARTED);
  spinner.start();
  ros::waitForShutdown();
  Stop();
  ReportModuleStatus(apollo::hmi::ModuleStatus::STOPPED);
  AINFO << Name() << " exited.";
  return 0;
}
```
- 其中的ros::AsyncSpinner spinner(1);是来自Ros中的线程管理，用以初始化1个线程
- status = Init() ，基类中没有实现Init()方法，status结果其实是模块中的Init()方法运行的结果。


模块的Init()方法:直接返回了成功码
```
modules\planning\planning.cc 模块的Init()方法

Status Planning::Init() {
    ...
    省略一部分代码
    ...
    CHECK(apollo::common::util::GetProtoFromFile(FLAGS_planning_config_file,
                                                &config_))
        << "failed to load planning config file " << FLAGS_planning_config_file;
    CheckPlanningConfig();
    ...
    省略一部分代码
    ...

    if (!AdapterManager::Initialized()) {
        AdapterManager::Init(FLAGS_planning_adapter_config_filename);
    }
}

```

初始化传入的参数：FLAGS_planning_adapter_config_filename实际上是利用Google开源库gflags宏：
```
DEFINE_string(planning_adapter_config_filename,
"modules/planning/conf/adapter.conf", 
"The adapter configuration file");

```

读取配置文件
```
CHECK(apollo::common::util::GetProtoFromFile(FLAGS_planning_config_file,
                                             &config_))
    << "failed to load planning config file " << FLAGS_planning_config_file;
if (!AdapterManager::Initialized()) {
  AdapterManager::Init(FLAGS_planning_adapter_config_filename);
}
```
其中配置文件的内容
```
modules/planning/config/adapter.config

config {
  type: LOCALIZATION
  mode: RECEIVE_ONLY
  message_history_limit: 1
}
.....//其中省略一部分代码//......
config {
  type: TRAFFIC_LIGHT_DETECTION
  mode: RECEIVE_ONLY
  message_history_limit: 1
}
is_ros: true
```
该文件表明，Planning模块配置以下几种类型的适配器：LOCALIZATION（仅接收）、CHASSIS（仅接收）、ROUTING_RESPONSE（仅接收）、ROUTING_REQUEST（仅发布）、PREDICTION（仅接收）、PLANNING_TRAJECTORY（仅发布）、TRAFFIC_LIGHT_DETECTION（仅接收），且使用ROS环境。

接下来回到AdapterManager::Init
上述代码调用了
AdapterManager::Init(FLAGS_adapter_config_filename)
```
modules\common\adapters\adapter_manager.cc

void AdapterManager::Init(const AdapterManagerConfig &configs) {
  if (Initialized()) {
    return;
  }

  instance()->initialized_ = true;
  if (configs.is_ros()) {
    instance()->node_handle_.reset(new ros::NodeHandle());
  }

  for (const auto &config : configs.config()) {
    switch (config.type()) {
      case AdapterConfig::POINT_CLOUD:
        EnablePointCloud(FLAGS_pointcloud_topic, config);
        break;
      case AdapterConfig::PLANNING_TRAJECTORY:
        EnablePlanning(FLAGS_planning_trajectory_topic, config);
        break;
      default:
        AERROR << "Unknown adapter config type!";
        break;
    }
  }
}  
```
上述代码中的FLAGS_chassis_topic、FLAGS_localization_topic、FLAGS_traffic_light_detection_topic、FLAGS_routing_request_topic、FLAGS_routing_response_topic、FLAGS_planning_trajectory_topic、FLAGS_prediction_topic是利用DEFINE_string宏定义出来的几个字符串变量，其缺省值分别为：
“/apollo/canbus/chassis”
“/apollo/localization/pose”
“/apollo/perception/traffic_light”
“/apollo/routing_request”
等等

代码中
EnableRoutingResponse、
EnablePlanning、
EnablePrediction，
是由REGISTER_ADAPTER (Planning) 定义的

命名空间Apollo::common::adapter内的适配器管理者类AdapterManager使用宏REGISTER_ADAPTER (Planning)注册了一个PlanningAdapter类对象及与其相关的变量、函数。

# 参考资料
1. Adapter适配器 http://www.fzb.me/apollo/adapter.html
2. https://blog.csdn.net/davidhopper/article/details/79176505 