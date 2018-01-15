---
title: "制作自己的Spring Boot Starter模块"
date: 2017-05-15T16:16:46+08:00
tags: [ "Java", "Spring Boot" ]
categories: [ "研发" ]
---

&#160; &#160;&#160;&#160;&#160;&#160;经过思考，放弃了自己搭建分布式文件系统的思路，决定采用第三方的文件服务器存储上传的图片。由于已经驻在阿里云上了，因此采用阿里的OSS。

&#160; &#160;&#160;&#160;&#160;&#160;阿里的SDK实在是没有体现出他们的水平，感觉就是大和杂，依赖很多，同样的功能都要引入好几个不同的包来处理，什么fastjson，gson，json-lib让人眼花。项目组伙伴吐槽后撂挑子了，只得去救火。

<!--more-->

&#160; &#160;&#160;&#160;&#160;&#160;做了个比较轻的包装，基本上只需配置，代码中不需要关注上传的动作了。考虑到项目中被伙伴嗯们广泛接受的Spring Boot，于是打开了Spring Boot 自带的Starter源代码，依样画葫芦也搞了Starter，总结下来有几个要点：  

1. **定义配置Propeties类**

    Propeties类一般定义为`XXXProperties`,其中`XXX`为配置的主题，是个普通的Java Bean。唯一的差别是用`ConfigurationProperties`对类进行了标注。如：
{{< highlight Java "linenos=inline" >}}
    @ConfigurationProperties(prefix = "ali.oss")
    public class OssUploadProperties {
        private String accessId;
        private String accessKey;
        private String endpoint;
        private String bucket;
        private Condition condition;
        private PolicyServer policyServer;
        private CallbackServer callbackServer;

        ......
    }
{{< /highlight >}}

2. **定义Configuration类**

    Configuration是个Java Config配置类，需使用`EnableConfigurationProperties`引入上述Properties类，在构造函数中完成注入。
{{< highlight Java "linenos=inline" >}}
    @Configuration
    @EnableConfigurationProperties({OssUploadProperties.class})
    public class OSSConfiguration {
        private final OssUploadProperties properties;

        public OSSConfiguration(OssUploadProperties properties) {
            this.properties = properties;
        }

        ......
    }
{{< /highlight >}}    

3. **编辑META-IN/spring.factories文件**

    这个文件就没的好说的了。
{{< highlight xml >}}
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=me.szlx.ali.OSSConfiguration
{{< /highlight >}}    

&#160; &#160;&#160;&#160;&#160;&#160;按照上述制作的Starter，有个缺陷就是，只要在Spring Boot环境中引入了，就会进行相关的初始化。如果要避免这点，可以先注释掉`META-IN/spring.factories`文件的相关内容，然后提供类似以下的标注类，并在需要启用的工程中用于标注主类即可。
{{< highlight Java "linenos=inline" >}}
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @Import(OSSConfiguration.class)
    public @interface EnableAliOSS {

    }
{{< /highlight >}}   