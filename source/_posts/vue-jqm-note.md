---
title: Vue 与 jQuery Mobile 混用
date: 2018-01-25 11:23:06
categories: 网页开发
tags:
    - Vue
    - jQuery Mobile
---
由于项目需要，在手机端定下的框架是 jQuery Mobile。但由于应用比较大，没有 MVVM 支持会越来越困难。
正好想尝试 Vue ，于是就直接开始 Vue 和 jQuery Mobile 混用的挖坑之路。
<!-- more -->
# 经验

## Detail 视图、表单自动生成

大纲-细节 (Master-Detail) 视图是经常用到的。每个对象的信息都不一样，而且往往会很多。我们这个项目中，起码有 20+ 的对象需要展示和填写。如果每个细节视图都像下面这样做，那工程量会非常大。

`HTML`
``` HTML
<ul id="content" data-role="listview" data-inset="false">
    <li>
        <b>联系人员</b>
        <span class="listview-aside">{{ item.call_person }}</span>
    </li>
    <li>
        <b>联系电话</b>
        <span class="listview-aside">{{ item.call_num }}</span>
    </li>
    <li>
        <b>签发日期</b>
        <span class="listview-aside">{{ item.call_date }}</span>
    </li>
</ul>
``` 
> 这样结合了 Vue，利用 Visual Studio Code 多点编辑的功能虽然也不慢，但是可以使用 Vue 寻求更简单的生成方式。

我们可以对这个页面所表示的对象建立一个类，比如就叫 `Call`，为这个类编写一个公有 `Array` 属性 `domMap`，表示将这个类映射到 DOM 元素上：

`JavaScript`
``` JavaScript
Call.prototype.domMap = [
    {
        key: "call_person",
        name: "联系人员",
        type: "text",
        hidden: false
    },{
        key: "call_num",
        name: "联系电话",
        type: "tel",
        hidden: false
    },{
        key: "call_date",
        name: "签发日期",
        type: "date",
        hidden: {
            listview: false,
            form: true
        }
    }
]
```

前端根据这个 `domMap` 自动生成 DOM 元素。

`HTML`
``` HTML
<ul data-role="listview" data-inset="true">
    <li v-for="(dommap, index) in detail.domMap" v-if="!dommap.hidden && !dommap.hidden.listview">
        <b>{{ dommap.name }}</b>
        <p v-if="dommap.type === 'textarea'" class="listview-aside">{{ detail[dommap.key] }}</p>
        <span v-else class="listview-aside">{{ detail[dommap.key] }}</span>
    </li>
</ul>
```
> `detail` 是 Vue 对象中的一个属性，即当前详情列表所表示的对象。

同理 `form` 也可以自动生成。碰到 `type` 是 `textarea` 的时候，生成一个 `<textarea>` 元素；其他情况下生成 `<input>` 元素，其 `type` 属性根据 `domMap` 中的 `type` 属性确定即可。

## 表单元素动态添加

甲方提出了在一个页面行动态添加行进行填写的要求，最后还要 Ajax 提交。
如果用 jQuery 的话，需要在 HTML 元素中 Append 一段 HTML 代码，
提交时循环查找所有行，然后组合成表单，最后提交。

但是用 Vue 的话，只需要在 HTML 上做好绑定，然后给被绑定的数组对象添加元素即可。
方便了很多。我们网页端是用纯 jQuery 编写的，移动端我用的 Vue。

`HTML`
``` HTML
<form id="index-form">
    <div v-for="(ind, index) in form.wInsRecordZs" class="nd2-card">
        <div class="card-title">
            <h3 class="card-primary-title"> 相关检测指数{{ index + 1 }} </h3>
        </div>
        <div class="card-supporting-text has-action has-title">
            <div class="ui-field-contain">
                <label v-bind:for="'zs_' + index">指数（单位）</label>
                <input type="text" name="zs" v-bind:id="'zs_' + index" value="" v-model="form.wInsRecordZs[index].zs"/>
            </div>
            <div class="ui-field-contain">
                <label v-bind:for="'value_' + index">值</label>
                <input type="text" name="value" v-bind:id="'value_' + index" value="" v-model="form.wInsRecordZs[index].value"/>
            </div>
            <div class="ui-field-contain">
                <label v-bind:for="'stand_' + index">标准</label>
                <input type="text" name="stand" v-bind:id="'stand_' + index" value="" v-model="form.wInsRecordZs[index].stand"/>
            </div>
        </div>
        <div class="card-action">
            <div class="row between-xs">
                <div class="col-xs-12 align-right">
                    <a href="#" class="ui-btn clr-primary ui-btn-inline" v-on:click="minusInticator(index)">删除 </a>
                </div>
            </div>
        </div>
    </div>
</form>
<div class="row between-xs">
    <div class="col-xs-12 align-center">
        <button href="#" class="ui-btn ui-btn-icon-left ui-btn-inline clr-primary" v-on:click="addIndicator()">添加检测指数 </button>
        <button href="#" class="ui-btn ui-btn-icon-left ui-btn-inline" v-on:click="clean()">清空 </button>
        <button class="ui-btn ui-btn-icon-left ui-btn-inline ui-btn-raised clr-primary" v-on:click="submit()"> 提交 </button>
    </div>
</div>
```

`JavaScript`
``` JavaScript
var vm = new Vue({
    el: "#index-form",
    data: {
        form: {
            wInsRecordZs: []
        }
    },
    methods: {
        addIndicator: function () {
            this.form.wInsRecordZs.push(new Indicator());
            setTimeout(function () {
                $("form").trigger("create")
            }, 10)
        },
        submit: function () {
            $.ajax({
                type: "POST",
                url: "http://127.0.0.1:3000/WInsR/save",
                data: mainpageVM.form.toForm().substr(1),
                success: function () {
                    new $.nd2Toast({
                        message : "提交成功", // Required
                    });
                },
                error: function() {
                    new $.nd2Toast({
                        message : "提交失败", // Required
                    });
                }
            });
        },
        clean: function () {
            mainpageVM.form.clean();
        },
        minusInticator: function (index) {
            mainpageVM.form.wInsRecordZs.splice(index, 1);
        }
    }
})
```

开发网页端的那个哥们表示：他跟不上我的进度了。

# 踩坑

## 表单元素

对于`input`类型是`text` `tel` `number`这些需要输入的表单元素，没有什么问题。
使用第三方插件提供支持的`date` `time` `datetime`类型元素，也没有问题。
但是对于`radio` `checkbox`这两个元素，实测 jQuery Mobile 和 Vue 无法自动结合。

例如

`HTML`
``` HTML
<fieldset id="fieldset" data-role="controlgroup">
    <legend>项目状态</legend>
    <label><input v-model="form.project_state" type="radio" name="project_state" value="在建"/>在建</label>
    <label><input v-model="form.project_state" type="radio" name="project_state" value="改扩建"/>改扩建</label>
    <label><input v-model="form.project_state" type="radio" name="project_state" value="生产"/>生产</label>
    <label><input v-model="form.project_state" type="radio" name="project_state" value="停产"/>停产</label>
</fieldset>
```

`JavaScript`
``` JavaScript
var content = new Vue({
    el: "fieldset",
    data: {
        form: {
            project_state: ""
        }
    }
})
```

这种情况下，进行单选操作，`content.form.project_state`的值是不更改的，
有可能是 jQuery Mobile 在实现的时候，阻断了事件的传播，Vue 无法获取到真正的值。
如果希望更改 Vue 对象中的值，那么需要给每个按钮元素加上`onclick`事件响应函数，替换`v-model`绑定。

`HTML`
``` HTML
<fieldset id="fieldset" data-role="controlgroup">
    <legend>项目状态</legend>
    <label><input v-model="form.project_state" type="radio" name="project_state" onclick="content.form.project_state= '在建'" value="在建"/>在建</label>
    <label><input v-model="form.project_state" type="radio" name="project_state" onclick="content.form.project_state= '改扩建'" value="改扩建"/>改扩建</label>
    <label><input v-model="form.project_state" type="radio" name="project_state" onclick="content.form.project_state= '生产'" value="生产"/>生产</label>
    <label><input v-model="form.project_state" type="radio" name="project_state" onclick="content.form.project_state= '停产'" value="停产"/>停产</label>
</fieldset>
```

现在这样就可以更改`content`中的值了。

但是有趣的是，如果你使用了`v-model`绑定元素到某个对象（例如这里的`project_state`），
那么当你在初始化 Vue 对象的时候，对该对象（`project_state`）赋值，
绑定到它的单选按钮组会自动根据该对象的值进行初始化。
比如你设置了`project_state`的初始值为“生产”，
那么“生产”对应的第三个单选按钮会在文档初始化的过程中被选中。

## 动态加载 DOM 元素

按照 jQuery Mobile 官网上的示例，所有标签的样式在编写 HTML 文档的时候，
都使用 `data-` 属性确定，文档加载完毕后会自动初始化并加载样式。
当在文档中使用 Vue 动态添加元素的时候，就会出现没有样式的情况。

解决这个问题的办法有两个。

1. **如果元素只需要使用 jQuery Mobile 的样式**，在使用 `v-for` 的时候，直接指定元素的 `class` 属性，同时也加上 `data-` 属性。
2. 如果元素需要 jQuery Mobile 自带的一些事件响应（比如 `collapsible` 元素，点击标头需要可以展开或折叠其内容。这时，在添加元素事件发生后，在其父元素上触发`create`事件。例如：我在表单中根据 Vue 绑定值 `form.wInsRecordZs[]` 动态添加一个 `collapsible`，就可以这样做
    ``` JavaScript
    this.form.wInsRecordZs.push(new Indicator());
    setTimeout(function() {
        $("form").trigger("create")
    }, 10)
    ```
    其中 `trigger("create")` 方法可以让 jQuery Mobile 对添加的元素自动初始化。设置一段时间后再触发是实践的结果，绑定的数组添加后，DOM 元素不一定立刻添加了，需要等一小段时间才能添加。我们设置一段延迟，保证在 DOM 元素已经添加之后，再触发 `create` 事件。


