# OTA单分区差分和全量升级示例

环境说明：

- 硬件环境：rk3568
- 系统环境：OpenHarmony 4.1.1

本文档主要介绍在OpenHarmony中如何验证单分区全量升级和差分升级。包括环境搭建及问题案例说明。

## 1 工程配置

### 1.1 配置config.json

路径：'\\vendor\openvalley\rk3568\config.json'

- enabl_absystem

        "enable_absystem": false,

- 关闭selinux
  
在验证阶段先关闭selinux，产品化时，再添加相应写文件权限

        "selinux": false,

## 2 升级包制作

OTA升级包制作工具路径：'\\base\\update\\packaging_tools'

其中包括全量其中包括全量升级包、差分升级包、流式升级包制作。

- 全量升级包：将所有目标版本的镜像均通过全量镜像的方式打包获得的升级包。
- 差分升级包：将目标版本与当前版本的差异通过差分镜像的方式打包获得的升级包。
- 流式升级包：将目标版本与当前版本的差异通过流式镜像的方式打包获得的升级包。

### 2.1 目录介绍
