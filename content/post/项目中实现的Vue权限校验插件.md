---
title: "一个项目中实现的Vue权限校验插件"
date: 2017-07-10T16:40:43+08:00
tags: [ "JavaScript", "前端", "Vue" ]
categories: [ "研发" ]
---

&#160; &#160;&#160;&#160;&#160;&#160;阅读[Vue文档](https://cn.vuejs.org/)知道可以自定义插件和指令后便跃跃欲试，但一直没有尝试。最近项目进度压力不大，便尝试着使用Vue[自定义插件](https://gitee.com/lxrj/Vue-Auth/blob/master/Authorization.js)和指令将项目权限校验部分进行了重构。

<!--more-->

&#160; &#160;&#160;&#160;&#160;&#160;首先是包装用户权限检测的功能。思路非常简单，就是在用户的权限数据中进行检测指定的角色或权限，检测到了则说明有权限，否则说明没有权限。用户的权限数据由`Role`和`Permission`数组组成，在权限机制初始化的时候提供，也可以提供返回它们的函数。初始化如下：
{{< highlight Javascript "linenos=inline" >}}
Vue.use(Authorization, {
  authorization (context) {
    let roles = []
    let permissions = []
    ......
    return {roles, permissions}
  },
  options: {
    context: ......
  }
})
{{< /highlight >}}

&#160; &#160;&#160;&#160;&#160;&#160;其次是实现自定义的指令`auth`，使用后如果有权限则渲染后出指令所在的元素，否则忽略掉指令所在元素。
{{< highlight Javascript "linenos=inline" >}}
    Vue.directive('auth', {
      bind: function (el, binding) {
        let checkFunction = _authorization.hasPermission
        if (binding.arg === 'role') {
            checkFunction = _authorization.hasRole
        }
        if (!binding.arg && binding.arg !== '' && binding.arg !== 'permission') {
          throw new Error('指令参数错误')
        }
        if (!checkFunction(binding.value)) {
          el.parentNode.removeChild(el)
        }
      }
    })
{{< /highlight >}}

&#160; &#160;&#160;&#160;&#160;&#160;使用方式如下：
{{< highlight html "linenos=inline" >}}
    <div v-auth:role="aRole"> 
        ......
    </div>

    <!--或-->
    <div v-auth:permission="aPermission"> 
        ......
    </div>

    <!--或-->
    <div v-auth="aPermission"> 
        ......
    </div>
{{< /highlight >}}