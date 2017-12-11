---
layout:     post
title:      SpringBoot集成FastDFS
subtitle:   
date:       2017-12-12
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - Spring Boot 
    - FastDFS
---


### FastDFS是什么

FastDFS是一个开源的轻量级分布式文件系统，它对文件进行管理，功能包括：文件存储、文件同步、文件访问（文件上传、文件下载）等，解决了大容量存储和负载均衡的问题。特别适合以文件为载体的在线服务，如相册网站、视频网站等等。

FastDFS为互联网量身定制，充分考虑了冗余备份、负载均衡、线性扩容等机制，并注重高可用、高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务。

### SpringBoot集成FastDFS

假如已经有FastDFS服务器，怎么与FastDFS集成？

* 新建服务（必须是Springboot项目）添加依赖[`fastdfs-client`](https://github.com/tobato/FastDFS_Client)

```
<dependency>
    <groupId>com.github.tobato</groupId>
    <artifactId>fastdfs-client</artifactId>
    <version>1.25.2-RELEASE</version>
</dependency>

<!-- Test -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.14</version>
</dependency>
```

* 将FastDFS配置引入项目

仅需要在启动类上增加注解 `@EnableSwagger2 `，启动该注解使得用在Controller中的Swagger注解生效

```
@Import(FdfsClientConfig.class)
@SpringBootApplication
public class Server {

    public static void main(String[] args) {
        SpringApplication.run(Server.class, args);
    }
}
```

* 增加FastDFS配置

```
fdfs.soTimeout=1500
fdfs.connectTimeout=600
#缩略图生成参数
fdfs.thumbImage.width=150
fdfs.thumbImage.height=150
#TrackerList参数,支持多个
fdfs.trackerList[0]=10.0.0.112:22122
#fdfs.trackerList[1]=10.0.0.112:22122
```

* 将FastDFS制作成SpringBoot组件

在目前的项目当中将FastDFS主要用于图片存储，基于FastFileStorageClient接口和SpringMVC提供的MultipartFile接口封装了一个简单的工具类

```
package com.gemantic.fs;

import com.github.tobato.fastdfs.domain.StorePath;
import com.github.tobato.fastdfs.exception.FdfsUnsupportStorePathException;
import com.github.tobato.fastdfs.service.FastFileStorageClient;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.FilenameUtils;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.nio.charset.Charset;

/**
 * @author Yezhiwei
 * @date 17/12/11
 */
@Slf4j
@Component
public class FastDFSClientWrapper {


    @Autowired
    private FastFileStorageClient storageClient;


    /**
     * 上传文件
     *
     * @param file 文件对象
     * @return 文件访问地址
     * @throws IOException
     */
    public String uploadFile(MultipartFile file) throws IOException {
        StorePath storePath = storageClient.uploadFile(file.getInputStream(), file.getSize(), FilenameUtils.getExtension(file.getOriginalFilename()), null);
        return getAccessUrl(storePath);
    }

    /**
     * 将一段字符串生成一个文件上传
     *
     * @param content       文件内容
     * @param fileExtension
     * @return
     */
    public String uploadFile(String content, String fileExtension) {
        byte[] buff = content.getBytes(Charset.forName("UTF-8"));
        ByteArrayInputStream stream = new ByteArrayInputStream(buff);
        StorePath storePath = storageClient.uploadFile(stream, buff.length, fileExtension, null);
        return getAccessUrl(storePath);
    }

    private String getAccessUrl(StorePath storePath) {
        String fileUrl = storePath.getFullPath();
        return fileUrl;
    }

    /**
     * 删除文件
     *
     * @param fileUrl 文件访问地址
     * @return
     */
    public void deleteFile(String fileUrl) {
        if (StringUtils.isEmpty(fileUrl)) {
            return;
        }
        try {
            StorePath storePath = StorePath.praseFromUrl(fileUrl);
            storageClient.deleteFile(storePath.getGroup(), storePath.getPath());
        } catch (FdfsUnsupportStorePathException e) {
            log.warn(e.getMessage());
        }
    }
}

```

>
客户端主要包括以下接口： 
TrackerClient - TrackerServer接口 
GenerateStorageClient - 一般文件存储接口 (StorageServer接口) 
FastFileStorageClient - 为方便项目开发集成的简单接口(StorageServer接口) 
AppendFileStorageClient - 支持文件续传操作的接口 (StorageServer接口)

注：可根据自身的业务需求，将其它接口也添加到组件类中即可。

* 测试

```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
@Slf4j
public class FastDFSClientTest {

    @Autowired
    private FastDFSClientWrapper fastDFSClientWrapper;

    @Test
    public void updateFileByContent() {


        String url = fastDFSClientWrapper.uploadFile("This is test", "png");
        log.info("url {}", url);
        
        //输出/group112/M00/BB/6D/CgAAcFoukU2sCPUsAAAADDzy24o518.png
    }


    @Test
    public void deleteFile() {


        fastDFSClientWrapper.deleteFile("http://10.0.0.112/group112/M00/BB/6D/CgAAcFoukU2sCPUsAAAADDzy24o518.png");
    }
}
```











