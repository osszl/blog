---
title: "Validation库使用指南"
date: 2017-08-09T16:22:11+08:00
tags: [ "Java" ]
categories: [ "研发" ]
---

&#160; &#160;&#160;&#160;&#160;&#160;[Validation工具库](https://gitee.com/lxrj/validation/)是个为改进团队协作，规范代码而生的工具包，主要应用场景是业务的前置条件检测以及后置条件检测。如果检测不通过则立即抛出异常并终止处理。它具有以下主要特点：  

> 1. 简单但不失灵活，开箱即用；
> 2. 友好并可配置；  
> 3. 基本依赖只有slf4j-api,其他依赖根据使用方式可选：[fast-classpath-scanner](https://github.com/lukehutch/fast-classpath-scanner)，[snakeyaml](https://bitbucket.org/asomov/snakeyaml)，[config](https://github.com/lightbend/config)。

<!--more-->

### 基本用法

&#160; &#160;&#160;&#160;&#160;&#160;为了方便，假设有某系统中有以下对象：
{{< highlight Java "linenos=inline" >}}
        public class Address {
            private String province;
            private String city;
            private county;
            private detailed;
            
            ......
        }

        publi class Personnel {
            private String name;
            private Address home;
            private Address office;

            ......
        }
{{< /highlight >}}

&#160; &#160;&#160;&#160;&#160;&#160;在业务处理中要求`Personnel`的名称、家庭地址和办公地址都不能为空。使用[Validation工具库](https://gitee.com/lxrj/validation/)就是：
{{< highlight Java "linenos=inline" >}}
    String nameNotNull = "PERSONNEL_NAME_NOT_NULL";
    String adddressNotNull = "PERSONNEL_ADDRESS_NOT_NULL";

    // 这样校验
    Constraint.of(nameNotNull).validator().invalidIf(personnel.getName() != null);
    Constraint.of(adddressNotNull).validator().invalidIf(personnel.getHome() != null);
    Constraint.of(adddressNotNull).validator().invalidIf(personnel.getOffice() != null);

    // 或者
    Validators.invalidIf(personnel.getName() != null, nameNotNull);
    Validators.invalidIf(personnel.getHome() != null, adddressNotNull);
    Validators.invalidIf(personnel.getOffice() != null, adddressNotNull);

    // 或者
    Validators.of(nameNotNull).invalidIf(personnel.getName() != null);
    Validators.of(adddressNotNull).invalidIf(personnel.getHome() != null);
    Validators.of(adddressNotNull).invalidIf(personnel.getOffice() != null);


    // 重用校验器还可以写成
    Validator nameValidator = Validators.of(nameNotNull);
    Validator adddressValidator = Validators.of(adddressNotNull);
    nameValidator.invalidIf(personnel.getName() != null);
    adddressValidator.invalidIf(personnel.getHome() != null);
    adddressValidator.invalidIf(personnel.getOffice() != null);
{{< /highlight >}}

&#160; &#160;&#160;&#160;&#160;&#160;当条件成立时，`Validator`将抛出异常。异常的信息来自`Constraint`，包含其代码和描述。在上面展示的代码中，系统首先尝试将传入的`Constraint字符串参数`当成约束代码去匹配受管理的约束（包括工具包内置的，或程序定义和配置的），如果匹配失败，则以`Constraint.UNCODED.code()`为代码，`Constraint字符串参数`为描述构建临时的`Constraint`。

&#160; &#160;&#160;&#160;&#160;&#160;与`Validator`同理，当`Constraint`反复使用时，更合理的方式是事先构造好`Constraint`取代`Constraint字符串参数`，譬如：
{{< highlight Java "linenos=inline" >}}
    Constraint adddressConstraint = Constraint.of(adddressNotNull);
    adddressConstraint.validator().invalidIf(personnel.getHome() != null);
    adddressConstraint.validator().invalidIf(personnel.getOffice() != null);
{{< /highlight >}}

&#160; &#160;&#160;&#160;&#160;&#160;通常情况面，重用`Validator实例`是最佳的方式，应为其内部已经重用了`Constraint`。

&#160; &#160;&#160;&#160;&#160;&#160;真的很简单，不是吗？是的，使用很简单。但是不为空的逻辑实际情况可能会复杂得多。譬上例中，仅仅检测adddressNotNull其实并不能准确的表达业务约束。如果地址不为空，但是其内容为空，更进一步，其地址必须要能正确的表达业务（试想有个地址为“地球省亚洲市喜马拉雅县唐古拉公园”会怎么样）。针对这类情形，[Validation工具库](https://gitee.com/lxrj/validation/)提供了可选的支持，那就是用一个`Conditional实例`去替换上述中的布尔值：

{{< highlight Java "linenos=inline" >}}
    Predicate<Address> isBadAddress = address -> {
        ......
    }

    Conditional homeIsBad = Conditional.newPredicative(isBadAddress, personnel.getHome());
    Conditional officeIsBad = Conditional.newPredicative(isBadAddress, personnel.getOffice());
        
    adddressValidator.invalidIf(homeIsBad);   
    adddressValidator.invalidIf(officeIsBad);   
{{< /highlight >}}

&#160; &#160;&#160;&#160;&#160;&#160;将“校验逻辑”放到`Address`中，直接`Address::invalid()`不是更好吗？是的，某些情况这样会更好，这里强调的是这样做不好的场景：  

> 1. 在初期的业务场景中，不需要校验逻辑；
> 2. 在不同的业务场景中，“校验逻辑”是不同的；
> 3. 在不同的业务场景中，校验“违规信息”有不同的。

&#160; &#160;&#160;&#160;&#160;&#160;[Validation工具库](https://gitee.com/lxrj/validation/)专门设计了“名称空间”对约束的代码进行包装来处理此类问题。包装的方法也很简单：用名称空间的代码 + “.” + 约束代码替换原始的约束代码。而检索约束时，则是从当前名称空间开始，逐层往上层名称空间搜索，一旦搜索到则立即返回，从而保证提供的违规描述信息最贴近当前场景的。

&#160; &#160;&#160;&#160;&#160;&#160;使用“名称空间”能为校验违规提供更符合场景的信息。“名称空间”的用法如下：
{{< highlight Java "linenos=inline" >}}
        NameSpace homeScene = NameSpace.of("scene.home")
        
        homeScene.validator(adddressNotNull).invalidIf(homeIsBad);
        // 或者
        homeScene.invalidIf(homeIsBad, adddressNotNull);
        // 或者
        adddressValidator.invalidIf(homeIsBad, homeScene);
{{< /highlight >}}
&#160; &#160;&#160;&#160;&#160;&#160;嗯，约束的校验逻辑与违规信息分离了，似乎很美好。

### Constraint配置

&#160; &#160;&#160;&#160;&#160;&#160;[Validation工具库](https://gitee.com/lxrj/Validation/)是开箱即用的，即使不做任何配置，也能工作起来。但灵活配置时其一个核心特征。合理配置可以得到意想不到的惊喜：

> 1. 编码过程中获得IDE友好支持；
> 2. 实现违规信息的提供与使用的团队成员角色分工；
> 3. 部署时实现违规信息的替换；
> 4. 运行时实现违规信息的替换；
> 5. 替换违规行为的默认处理方式；
> 6. 替换名称空间逻辑。 

&#160; &#160;&#160;&#160;&#160;&#160;[Validation工具库](https://gitee.com/lxrj/Validation/)支持不同颗粒度的配置：约束系统，约束容器，维护处理，约束，名称空间。下面介绍使用过程中用的比较多的“约束级别”的配置。
{{< highlight Java "linenos=inline" >}}
    String packageName = "me.szlx.constraint";
    ImplementBundle.from(packageName).bindTo(ConstraintSystem.get().getConstraintContainer());
{{< /highlight >}}
&#160; &#160;&#160;&#160;&#160;&#160;其它高级的配置可以使用`Configurer`工具类进行配置。工具包内置的`Bundle`有:

> 1. **ImplementBundle：**  通过扫描类路径下实现了接口`Constraint`的实例或类（非枚举类需无参构造函数）查找约束；
> 2. **JdbcBundle：**  从数据表中查找约束；
> 3. **MapBundle：**  从Map的键值对中提取约束；
> 4. **PropertiesBundle：**  从属性文件中提取约束；
> 5. **TypeSafeBundle：**  从[config](https://github.com/lightbend/config)支持的格式文件中提取约束；
> 6. **YamlBundle：**  从yaml格式文件中提取约束。

### 最佳实践

1. **使用枚举定义约束**

    使用枚举类定义的约束可以获得IDE环境的语法支持。不要试图将所有的约束放在同一个枚举类中。
{{< highlight Java "linenos=inline" >}}
    public enum PersonnelConstraint implements Constraint {
        NAME_NOT_NULL("nameNotNull", "名称不能为空"), ADDDRESS_NOT_NULL("adddressNotNull", "地址不能为空");
    
        private String code;
        private String brief;
    
        PersonnelConstraint(String code, String brief) {
            this.code = code;
            this.brief = brief;
        }
    
        @Override
        public String code() {
            return code;
        }
    
        @Override
        public String brief() {
            return brief;
        }
    }
{{< /highlight >}}

2. **使用名称空间组织场景化的信息**

    名称空间可以有效组织约束违规描述，这样提供贴近场景的描述提供了可能。名称空间支持多层次的父子级联。
{{< highlight Java "linenos=inline" >}}
    public enum SCENE implements NameSpace {
        CONSTRAINTS, CONSTRAINT, PERSONNEL;
    }
{{< /highlight >}}
    以上使用枚举类实现了枚举名称空间。枚举类实现的枚举空间的字面代码前缀格式为：短类名 + “.” + 枚举变量名。但`CONSTRAINTS`和`CONSTRAINT`这两个名称做了特殊处理，前缀格式直接为：短类名。因此上面的枚举类实际定义了2个名称空间前缀：`SCENE`和`SCENE.PERSONNEL`。

    还可以定义多层次名称空间：
{{< highlight Java "linenos=inline" >}}
    public interface SCENE {
        NameSpace PERSONNEL = NameSpace.of("PERSONNEL");
        NameSpace PERSONNEL_NAME = NameSpace.of("NAME", PERSONNEL);
        NameSpace PERSONNEL_ADDRESS = NameSpace.of("ADDRESS", PERSONNEL);
    }
{{< /highlight >}}
    以上接口定义了3个名称空间：`PERSONNEL`，`PERSONNEL_NAME`和`PERSONNEL_ADDRESS`。其中后两者是前者的子名称空间，`PERSONNEL`名称空间的前缀为“PERSONNEL”，`PERSONNEL_NAME`名称空间的前缀为“PERSONNEL.NAME”，`PERSONNEL_ADDRESS`名称空间的前缀为“PERSONNEL.ADDRESS”。子名称空间的全前缀为：子名称空间的全前缀 + “.” + 子名称空间的前缀。

    以下几种方式的效果相同的：
{{< highlight Java "linenos=inline" >}}
    boolean predicate = personnel.getName() != null;
    Validators.invalidIf(predicate, "PERSONNEL.NAME.NOT_NULL");
    Validators.invalidIf(predicate, "NAME.NOT_NULL", PERSONNEL);
    Validators.invalidIf(predicate, "NOT_NULL", PERSONNEL_NAME);
{{< /highlight >}}

    使用名称空间组织场景时，尽管可以，但是建议约束代码不要使用点（.）分的字符串，以免与名称空间混淆。使用点分约束代码时，应当右边第一个“.”右边的字符串看成是约束代码，而左边的则当成是名称空间代码。


