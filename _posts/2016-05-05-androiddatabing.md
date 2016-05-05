---
title:android databinding使用中遇到的坑
---
记录下使用androiddatabinding中遇到的坑：项目中使用时 DataBindingUtil.setContentView方法始终返回Nullpointerexception,查了各种资料，最开始看到这个方法的注释中说引用的layout可能是使用include导致的，但是后来验证了不是（官方说法应该是使用merge会有可能导致问题），最终发现是因为基类BaseActivity复写了setContentView方法。

* 说明：项目配置；classpath 'com.android.tools.build:gradle:2.1.0-beta3'