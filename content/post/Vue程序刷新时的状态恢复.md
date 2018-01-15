---
title: "Vue程序刷新时的状态恢复"
date: 2017-02-09T13:19:15+08:00
tags: [ "JavaScript", "Vue" ]
categories: [ "研发" ]
---

&#160; &#160;&#160;&#160;&#160;&#160;因为[OSChina](http://www.oschina.net/)的一篇介绍 Element UI 的文章，跟到官网看了下提供的组件合文档，便起了蹭了一回[Vue](https://vuejs.org/)的热度。接着看了一天[Vue中文文档](https://cn.vuejs.org/)便铁定了心。

&#160; &#160;&#160;&#160;&#160;&#160;呼哧呼哧地花了3天时间，搭了个单页程序的工程架子，完成了登录，权限校验和个人信息等功能的开发做为demo后，便交给团队其它伙伴们了。

<!--more-->

&#160; &#160;&#160;&#160;&#160;&#160;开发过程中，伙伴们碰到了很多的疑难，基本上都是知识结构欠缺以及不愿意阅读文档的原因，直到有人说不能按F5为止。

&#160; &#160;&#160;&#160;&#160;&#160;收到反馈后，第一反应是还真有问题。因为搭建工程架子的过程中，出于习惯的做法，将系统公用基础数据，以及用户登录后鲜少变动的个人基础数据缓存在前端的单页程序里。这样做的理由是：

> 1. **减少网络请求频次：**    
>    将个人数据缓存，可免去不必要的ajax请求。如登录用户的基本信息和角色等。
> 2. **更少网络数据传输：**    
>    ajax请求时，原本需“代码名称”值对一起传输为只传输“代码”。  
> 3. **后台编码逻辑简化**   
>    后台服务业务处理免去提供名称的职责，省去了为了提供名称而产生的关联，或名称填充步骤。

如果这些缓存的数据丢失，程序行为异常时必然的。而按F5浏览器刷将销毁现有页面，重建当前URL指待的页面。显然地，构建前端页面数据缓存只会在特殊的环节进行，而不能再每个页面中进行。否则@#￥%，实在说不下去，不说了。    

&#160; &#160;&#160;&#160;&#160;&#160;由于之前阅读文档 Vue Router 给了我极其深刻的印象，感觉其钩子能干这事情。于是开始撸代码，测试。嗯，前进，后退，日志打印出来了，逻辑符合预期，于是F5。啥？日志没了，程序悲催了？检查代码，貌似没低级错误，为啥不行呢？

&#160; &#160;&#160;&#160;&#160;&#160;折腾过后，痛定思痛，继续看文档，对着[Vue实例生命周期图](https://cn.vuejs.org/v2/guide/instance.html)发呆，继续发呆，最后觉定在页面的根组件的`beforeCreate`里试试运气。

{{< highlight JavaScript "linenos=inline" >}}
    beforeCreate () {
    let redirectTarget = (target) => {
      let fullPath = this.$route.fullPath
      return fullPath !== '/home' ? `${target}?target=${fullPath}` : target
    }
    let prepare = (isLogin) => {
      if (isLogin) {
        if (this.shouldChangePassword) {
          this.$router.replace(redirectTarget('/profile/password/enhancement'))
        } else {
          refreshCache(this.$store)
        }
      } else {
        this.$router.replace(redirectTarget('/login'))
      }
    }
    if (!this.logined) {
      retrieveAccountInfo(this.$store).then(isOk => prepare(isOk)).catch(() => {
        this.$router.replace(`/login?target=${this.$route.path}`)
      })
    } else {
      prepare(false)
    }
  }

{{< /highlight >}}

> 上面代码中包含了额外的“强制密码修改控制”，“访问控制登录控制”，这些特征与本文主题没直接关系。`retrieveAccountInfo`中有自动登录实现逻辑，自动登录失败后转显示登录。

&#160; &#160;&#160;&#160;&#160;&#160;提心调单的F5，嗯，貌似可以，再F5，貌似还可以，继续F5……最后，F5对我就风轻云淡了。