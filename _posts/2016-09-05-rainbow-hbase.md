---
layout: post
title:  "Hbase彩虹表设计"
categories: HBase
tags:  rainbow hbase
---

* content
{:toc}

## Hbase彩虹相关设计

### MEID
 - 前提 ：环境为保德集群(4个HRegionServer(24cores&64Rom)),数据量约6T(5.5T)
 - 设计 ：
   1. 4个HRegionServer平均分担6T的数据，每个HRegionServer平均分担1.5T数据





   2. HRegionServer下每个Store大小为10G，每个HRegionServer管理的HRegion为150（1.5T/10G）个，所有HRegion为600（150*4）个
 - 相关参数配置 ：

|Property|Value|
|------------- |:-------------:|
|hbase.hregion.max.filesize|10G|
|hbase.hstore.blockingStoreFiles|10|
|hbase.hregion.memstore.flush.size|256|
|hfile.block.cache.size|0.2|
|hbase.regionserver.global.memstore.upperLimit/lowerLimit|0.4/0.38|
|HBase Master 的 Java 堆栈大小（字节）|4G|
|HBase RegionServer 的 Java 堆栈大小（字节）|4G|

  - 表设计（高表）

| | FamilyColumn|
|------------- |:-------------:|-------------:|
|Key|indentiy(Column)|-- | 
|575b7bfc9f51fde2fa2bb3599e29ec84（MD5密文）|1：863564021100236(Column：MD5明文)|--|
   
  - 建表语句

```
create 'meidrainbow','identity',{ NUMREGIONS => 600, SPLITALGO => 'HexStringSplit' }
```

   - 其他
    - 时间版本设置为1
    - key为MD5值，天然均匀分布
    - memstore可根据实际服务器配置调整
    - BlockCache采用默认的64M,可根据实际服务器配置调整
   - 时间复杂度
     - 如果它还在MemStore里

       O(1) 查找region  +  O(log n)用来在MemStore里定位KeyValue (n=平均一个HFile里KeyValue条目的数量)
     - 如果它不在MemStore里

       O(1) 查找region  +  O(log m)用来在HFile里查找正确的数据块 (m=HFile中的数据块（HFile Block）的数量) + O(K)(K=块中KeyValue条目的数量)

       
   - 测试（MEID RainBow）
     - 总记录数：113568996276
     - 响应时间：小于1s
   - 部分截图如下



![1](http://120.236.169.50:88/imc/userprofile/uploads/4de188a3da491c7cce3d4465e74e52e3/1.png)


![2](http://120.236.169.50:88/imc/userprofile/uploads/5782e979d59830ba99e6624766be25d1/2.png)


![3](http://120.236.169.50:88/imc/userprofile/uploads/d0cd308e22a00bb25b369e5a8d6b3441/3.png)


![4](http://120.236.169.50:88/imc/userprofile/uploads/845e5d8df38c8b51dfe9b8b61af2a7c8/4.png)
