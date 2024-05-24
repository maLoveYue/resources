# Layui介绍



Layui 是一套免费的开源 Web UI 组件库，采用自身轻量级模块化规范，遵循原生态的 HTML/CSS/JavaScript 开发模式，极易上手，拿来即用。

[layui官网](https://layui.dev/docs/2/)

# 布局

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>layout 管理界面大布局示例 - Layui</title>
  <meta name="renderer" content="webkit">
  <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link href="//cdn.staticfile.org/layui/2.9.7/css/layui.css" rel="stylesheet">
</head>
<body>
<div class="layui-layout layui-layout-admin">
  <div class="layui-header">
    <div class="layui-logo layui-hide-xs layui-bg-black">layout demo</div>
    <!-- 头部区域（可配合layui 已有的水平导航） -->
    <ul class="layui-nav layui-layout-left">
      <li class="layui-nav-item layui-hide-xs"><a href="javascript:;">nav 1</a></li>
      <li class="layui-nav-item layui-hide-xs"><a href="javascript:;">nav 2</a></li>
      <li class="layui-nav-item layui-hide-xs"><a href="javascript:;">nav 3</a></li>
      <li class="layui-nav-item">
        <a href="javascript:;">nav groups</a>
        <dl class="layui-nav-child">
          <dd><a href="javascript:;">menu 11</a></dd>
          <dd><a href="javascript:;">menu 22</a></dd>
          <dd><a href="javascript:;">menu 33</a></dd>
        </dl>
      </li>
    </ul>
      
      
      <!-- 头部区域（可配合layui 已有的水平导航）  右侧导航栏 -->
    <ul class="layui-nav layui-layout-right">
      <li class="layui-nav-item layui-hide layui-show-sm-inline-block">
        <a href="javascript:;">
          <img src="//unpkg.com/outeres@0.0.10/img/layui/icon-v2.png" class="layui-nav-img">
          tester
        </a>
        <dl class="layui-nav-child">
          <dd><a href="javascript:;">Your Profile</a></dd>
          <dd><a href="javascript:;">Settings</a></dd>
          <dd><a href="javascript:;">Sign out</a></dd>
        </dl>
      </li>
      <li class="layui-nav-item" lay-header-event="menuRight" lay-unselect>
        <a href="javascript:;">
          <i class="layui-icon layui-icon-more-vertical"></i>
        </a>
      </li>
    </ul>
  </div>
   <!-- 头部区域（可配合layui 已有的水平导航） --> 
    
    
  <div class="layui-side layui-bg-black">
    <div class="layui-side-scroll">
      <!-- 左侧导航区域（可配合layui已有的垂直导航） -->
      <ul class="layui-nav layui-nav-tree" lay-filter="test">
        <li class="layui-nav-item layui-nav-itemed">
          <a class="" href="javascript:;">menu group 1</a>
          <dl class="layui-nav-child">
            <dd><a href="javascript:;">menu 1</a></dd>
            <dd><a href="javascript:;">menu 2</a></dd>
            <dd><a href="javascript:;">menu 3</a></dd>
            <dd><a href="javascript:;">the links</a></dd>
          </dl>
        </li>
        <li class="layui-nav-item">
          <a href="javascript:;">menu group 2</a>
          <dl class="layui-nav-child">
            <dd><a href="javascript:;">list 1</a></dd>
            <dd><a href="javascript:;">list 2</a></dd>
            <dd><a href="javascript:;">超链接</a></dd>
          </dl>
        </li>
        <li class="layui-nav-item"><a href="javascript:;">click menu item</a></li>
        <li class="layui-nav-item"><a href="javascript:;">the links</a></li>
      </ul>
    </div>
  </div>
    
    
  <div class="layui-body">
    <!-- 内容主体区域 -->
    <div style="padding: 15px;">
      <blockquote class="layui-elem-quote layui-text">
        Layui 框体布局内容主体区域
      </blockquote>
      <div class="layui-card layui-panel">
      
        
      </div>
      <br><br>
    </div>
  </div>
    
    
    
  <div class="layui-footer">
    <!-- 底部固定区域 -->
    底部固定区域
  </div>
</div>
 
<script src="//cdn.staticfile.org/layui/2.9.7/layui.js"></script>
<script>
//JS 
layui.use(['element', 'layer', 'util'], function(){
  var element = layui.element;
  var layer = layui.layer;
  var util = layui.util;
  var $ = layui.$;
  
  //头部事件
  util.event('lay-header-event', {
    menuLeft: function(othis){ // 左侧菜单事件
      layer.msg('展开左侧菜单的操作', {icon: 0});
    },
    menuRight: function(){  // 右侧菜单事件
      layer.open({
        type: 1,
        title: '更多',
        content: '<div style="padding: 15px;">处理右侧面板的操作</div>',
        area: ['260px', '100%'],
        offset: 'rt', // 右上角
        anim: 'slideLeft', // 从右侧抽屉滑出
        shadeClose: true,
        scrollbar: false
      });
    }
  });
});
</script>
</body>
</html>
```

# 栅格

为了丰富网页布局，简化 HTML/CSS 代码的耦合，并提升多终端的适配能力，layui 在 2.0 的版本中引进了自己的一套具备响应式能力的栅格系统。我们将容器进行了 12 等分，预设了 4*12 种 CSS 排列类，它们在移动设备、平板、桌面中/大尺寸四种不同的屏幕下发挥着各自的作用。

* layui-col-space10 列间距1-30
* layui-col-md4 以中型屏幕桌面为例，注意12等分即可
*  layui-col-md-offset4 列偏移

```html
<div class="layui-row layui-col-space10">
  <div class="layui-col-md4">
    1/3
  </div>
  <div class="layui-col-md4">
    1/3
  </div>
  <div class="layui-col-md4">
    1/3
  </div>
</div>


<div class="layui-row">
  <div class="layui-col-md4">
    4/12
  </div>
  <div class="layui-col-md4 layui-col-md-offset4">
    偏移4列，从而在最右
  </div>
</div>
```

# 字体图标

layui 的所有图标全部采用字体形式，取材于阿里巴巴矢量图标库（iconfont）。

```html
<i class="layui-icon layui-icon-face-smile"></i>   
```



# 卡片

一般的面板通常是指一个独立的容器，而折叠面板则能有效地节省页面的可视面积，非常适合应用于：QA说明、帮助文档等

```html
<div class="layui-card">
  <div class="layui-card-header">卡片面板</div>
  <div class="layui-card-body">
    卡片式面板面板通常用于非白色背景色的主体内<br>
    从而映衬出边框投影
  </div>
</div>
```

# 表单

```html

<form class="layui-form" action="">
  <div class="layui-form-item">
    <label class="layui-form-label">输入框</label>
    <div class="layui-input-block">
      <input type="text" name="title" required  lay-verify="required" placeholder="请输入标题" autocomplete="off" class="layui-input">
    </div>
  </div>
  <div class="layui-form-item">
    <label class="layui-form-label">密码框</label>
    <div class="layui-input-inline">
      <input type="password" name="password" required lay-verify="required" placeholder="请输入密码" autocomplete="off" class="layui-input">
    </div>
    <div class="layui-form-mid layui-word-aux">辅助文字</div>
  </div>
  <div class="layui-form-item">
    <label class="layui-form-label">选择框</label>
    <div class="layui-input-block">
      <select name="city" lay-verify="required">
        <option value=""></option>
        <option value="0">北京</option>
        <option value="1">上海</option>
        <option value="2">广州</option>
        <option value="3">深圳</option>
        <option value="4">杭州</option>
      </select>
    </div>
  </div>
  <div class="layui-form-item">
    <label class="layui-form-label">复选框</label>
    <div class="layui-input-block">
      <input type="checkbox" name="like[write]" title="写作">
      <input type="checkbox" name="like[read]" title="阅读" checked>
      <input type="checkbox" name="like[dai]" title="发呆">
    </div>
  </div>
  <div class="layui-form-item">
    <label class="layui-form-label">开关</label>
    <div class="layui-input-block">
      <input type="checkbox" name="switch" lay-skin="switch">
    </div>
  </div>
  <div class="layui-form-item">
    <label class="layui-form-label">单选框</label>
    <div class="layui-input-block">
      <input type="radio" name="sex" value="男" title="男">
      <input type="radio" name="sex" value="女" title="女" checked>
    </div>
  </div>
  <div class="layui-form-item layui-form-text">
    <label class="layui-form-label">文本域</label>
    <div class="layui-input-block">
      <textarea name="desc" placeholder="请输入内容" class="layui-textarea"></textarea>
    </div>
  </div>
  <div class="layui-form-item">
    <div class="layui-input-block">
      <button class="layui-btn" lay-submit lay-filter="formDemo">立即提交</button>
      <button type="reset" class="layui-btn layui-btn-primary">重置</button>
    </div>
  </div>
</form>
 
<script>
//Demo
layui.use('form', function(){
  var form = layui.form;
  
  //提交
  form.on('submit(formDemo)', function(data){
    layer.msg(JSON.stringify(data.field)); //data.field 表单的数据
    $.ajax({
        type: 'POST',
        url:" {% url 'index' %}",
        data: data.field,
        success: function (res){
            if (res.code == 0 ){
                layer.msg(res.msg,{'icon':6})
            }else {
                layer.msg(res.msg,{'icon':4})
            }
        },

        error: function (err){
            layer.msg(err,{'icon':4})
        }
    })
  });
});
</script>
```



# 上传

```html
{% extends 'base.html' %}

{% block title %}
    上传文件
{% endblock %}

{% block content %}

{#<button type="button" class="layui-btn" id="test1">#}
{#  <i class="layui-icon">&#xe67c;</i>上传图片#}
{#</button>#}

<div class="layui-btn-container">
  <button type="button" class="layui-btn layui-btn-normal" id="test1"> <i class="layui-icon">&#xe67c;</i>选择文件</button>
  <button type="button" class="layui-btn" id="test9">开始上传</button>
</div>

{% endblock %}


{% block js %}


<script>
layui.use(['upload'], function(){
  var upload = layui.upload;
  var $ = layui.$

  //执行实例
  var uploadInst = upload.render({
    elem: '#test1' //绑定元素
    ,url: "{% url 'upload' %}" // 上传接口，实际使用时改成您自己的上传接口即可。
    ,auto: false //关闭自动上传
    ,bindAction: '#test9' //绑定按钮提交
    ,data: {csrfmiddlewaretoken: $('input[name=csrfmiddlewaretoken]').val()}
    ,done: function(res){
      //上传完毕回调
      layer.msg(res.msg)
    }
    ,error: function(){
      //请求异常回调
    }
  });
});
</script>
{% endblock %}
```

# 数据表格

## 1. 方法渲染

```html
<table id="demo" lay-filter="test"></table>




var table = layui.table;
 
//执行渲染
table.render({
  elem: '#demo' //指定原始表格元素选择器（推荐id选择器）
  ,height: 315 //容器高度
  ,cols: [{}] //设置表头
  //,…… //更多参数参考右侧目录：基本参数选项
});
```

下面是目前 table 模块所支持的全部基础参数项一览表：

| 参数           | 类型               | 说明                                                         | 默认 / 用法                                                  |
| :------------- | :----------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| elem           | String/DOM         | 指定原始 table 容器的选择器或 DOM，方法渲染方式必填          | -                                                            |
| cols           | Array              | 设置表头。值是一个二维数组。方法渲染方式必填                 | [详见表头参数](https://layui.dev/2.7/docs/modules/table.html#cols) |
| url（系列）    | -                  | 异步数据接口相关参数。其中 url 参数为必填项                  | [详见异步参数](https://layui.dev/2.7/docs/modules/table.html#async) |
| data           | Array              | 直接赋值数据。既适用于只展示一页数据，也能对一段已知数据进行多页展示。 注：该参数与 url 参数只能二选一 | null                                                         |
| id             | String             | 设定容器唯一 id。id 是对表格的数据操作方法上是必要的传递条件，它是表格示例的重要索引，许多 API 都会用到它。若该参数未设置，则默认从 *<table id="test"></table>* 中的 id 属性值中获取。 | -                                                            |
| toolbar        | String/DOM/Boolean | 开启表格头部工具栏区域，该参数支持四种类型值：toolbar: '#toolbarDemo' *//指向自定义工具栏模板选择器*toolbar: '<div>xxx</div>' *//直接传入工具栏模板字符*toolbar: true *//仅开启工具栏，不显示左侧模板*toolbar: 'default' *//让工具栏左侧显示默认的内置模板* | false                                                        |
| defaultToolbar | Array              | 该参数可自由配置头部工具栏右侧的图标按钮                     | [详见头工具栏图标](https://layui.dev/2.7/docs/modules/table.html#defaultToolbar) |
| width          | Number             | 设定容器宽度。table容器的默认宽度是跟随它的父元素铺满，你也可以设定一个固定值，当容器中的内容超出了该宽度时，会自动出现横向滚动条。 | auto                                                         |
| height         | Number/String      | 设定容器高度                                                 | [详见 height](https://layui.dev/2.7/docs/modules/table.html#height) |
| cellMinWidth   | Number             | 全局定义所有常规单元格的最小宽度，一般用于列宽自动分配的情况。其优先级低于表头参数中的 minWidth | 60                                                           |
| lineStyle      | String             | 用于定义表格的多行样式，如每行的高度等。该参数一旦设置，单元格将会开启多行模式，且鼠标 hover 时会通过显示滚动条的方式查看到更多内容。 请按实际场景使用。 示例：`lineStyle: 'height: 95px;'` 注：v2.7.0 新增 | -                                                            |
| className      | String             | 用于给表格主容器追加 css 类名，以便更好地扩展表格样式 注：v2.7.0 新增 | -                                                            |
| css            | String             | 用于给当前表格主容器直接设定 css 样式，样式值只会对所在容器有效，不会影响其他表格实例。如： `css: '.layui-table-page{text-align: right;}'` 注：v2.7.0 新增 | -                                                            |
| escape         | Boolean            | 是否开启对内容的编码（转义 html）。注：v2.6.8 新增 注：从 v2.6.11 开始，默认开启。 | true                                                         |
| totalRow       | Boolean/String     | 是否开启合计行区域                                           | false                                                        |
| page           | Boolean/Object     | 开启分页。支持传入一个对象，里面可包含 [laypage](https://layui.dev/2.7/docs/modules/laypage.html#options) 组件所有支持的参数（jump、elem 除外） | false                                                        |
| pagebar        | String/DOM         | 用于开启分页区域的自定义模板，用法同 toolbar 注：v2.7.0 新增 | -                                                            |
| limit          | Number             | 每页显示的条数。值需对应 limits 参数的选项。 注意：*优先级低于 page 参数中的 limit 参数* | 10                                                           |
| limits         | Array              | 每页条数的选择项。 注意：*优先级低于 page 参数中的 limits 参数* | [10,…,90]                                                    |
| loading        | Boolean            | 是否显示加载条。若为 false，则在切换分页时，不会出现加载条。该参数只适用于 url 参数开启的方式 | true                                                         |
| scrollPos      | String             | 用于设定重载数据或切换分页时的滚动条的位置状态。 可选值如下：scrollPos: 'fixed' *//重载数据时，保持滚动条位置不变*scrollPos: 'reset' *//重载数据时，滚动条位置恢复置顶*若采用默认方式（*即重载数据或切换分页时，纵向滚动条恢复置顶，横向滚动条位置不变*），则无需设置改参数 注：v2.7.3 新增 | -                                                            |
| editTrigger    | String             | 是用于设定单元格编辑的事件触发方式。如双击：dblclick 注：v2.7.0 新增 | click                                                        |
| title          | String             | 定义 table 的大标题（在文件导出等地方会用到）                | -                                                            |
| text           | Object             | 自定义文本，如空数据时的异常提示等。                         | [详见自定义文本](https://layui.dev/2.7/docs/modules/table.html#text) |
| autoSort       | Boolean            | 默认 true，即直接由 table 组件在前端自动处理排序。 若为 false，则需自主排序，即由服务端返回排序好的数据 | [详见事件排序](https://layui.dev/2.7/docs/modules/table.html#onsort) |
| initSort       | Object             | 初始排序状态。 用于在数据表格渲染完毕时，默认按某个字段排序。 | [详见初始排序](https://layui.dev/2.7/docs/modules/table.html#initSort) |
| skin（系列）   | -                  | 设定表格各种外观、尺寸等                                     | [详见表格风格](https://layui.dev/2.7/docs/modules/table.html#skin) |
| before         | Function           | 数据拉取之前的回调。注：v2.7.3 新增                          | -                                                            |
| done           | Function           | 数据渲染完毕的回调。可借此做一些其它操作                     | [详见 done 回调](https://layui.dev/2.7/docs/modules/table.html#done) |
| error          | Function           | 数据请求失败的回调，返回两个参数：错误对象、内容 注：v2.6.0 新增 | -                                                            |

cols - 表头参数一览表

相信我，在你还尚无法驾驭 layui table 的时候，你的所有焦点都应放在这里，它带引领你完成许多可见和不可见甚至你无法想象的工作。如果你采用的是方法渲染，cols 是一个二维数组，表头参数设定在数组内；如果你采用的自动渲染，表头参数的设定应放在 *<th>* 标签上

| 参数          | 类型            | 说明                                                         | 默认 / 用法                                                  |
| :------------ | :-------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| field         | String          | 设定字段名。非常重要，且是表格数据列的唯一标识               | field: '昵称'                                                |
| title         | String          | 设定标题名称                                                 | title: 'nickname'                                            |
| width         | Number/String   | 设定列宽，若不填写，则自动分配；若填写，则支持值为：数字、百分比。 请结合实际情况，对不同列做不同设定。 | width: 200 width: '30%'                                      |
| minWidth      | Number          | 局部定义当前常规单元格的最小宽度，一般用于列宽自动分配的情况。其优先级高于基础参数中的 cellMinWidth | 60                                                           |
| type          | String          | 设定列类型。可选值有：normal（常规列，无需设定）checkbox（复选框列）radio（单选框列）numbers（序号列）space（空列） | -                                                            |
| LAY_CHECKED   | Boolean         | 是否全选状态（。必须复选框列开启后才有效，如果设置 true，则表示复选框默认全部选中。 | false                                                        |
| fixed         | String          | 固定列。可选值有：*left*（固定在左）、*right*（固定在右）。一旦设定，对应的列将会被固定在左或右，不随滚动条而滚动。 注意：*如果是固定在左，该列必须放在表头最前面；如果是固定在右，该列必须放在表头最后面。* | -                                                            |
| hide          | Boolean         | 是否初始隐藏列                                               | false                                                        |
| escape        | Boolean         | 当前列是否开启编码（转义 html），优先级大于基础参数项。 注：v2.7.5 新增 | true                                                         |
|               |                 |                                                              |                                                              |
| totalRow      | Boolean/String  | 是否开启该列的自动合计功能，默认：false。当开启时，则默认由前端自动合计当前行数据。从 layui 2.5.6 开始： 若接口直接返回了合计行数据，则优先读取接口合计行数据，格式如下：`</>{  "code": 0,  "totalRow": {    "score": "777"    ,"experience": "999"  },  "data": [{}, {}],  "msg": "",  "count": 1000}`如上，在 totalRow 中返回所需统计的列字段名和值即可。 另外，totalRow 字段同样可以通过 parseData 回调来解析成为 table 组件所规定的数据格式。从 layui 2.6.3 开始，如果 totalRow 为一个 string 类型，则可解析为合计行的动态模板，如：`</>totalRow: '{{= d.TOTAL_NUMS }} 单位'//还比如只取整：'{{= parseInt(d.TOTAL_NUMS) }}'` | true                                                         |
|               |                 |                                                              |                                                              |
| sort          | Boolean         | 是否允许排序。如果设置 true，则在对应的表头显示排序icon，从而对列开启排序功能。注意：*不推荐对值同时存在“数字和普通字符”的列开启排序，因为会进入字典序比对*。比如：*'张三' > '2' > '100'*，这可能并不是你想要的结果，但字典序排列算法（ASCII码比对）就是如此。 | false                                                        |
| unresize      | Boolean         | 是否禁用拖拽列宽。默认情况下会根据列类型（type）来决定是否禁用，如复选框列，会自动禁用。而其它普通列，默认允许拖拽列宽，当然你也可以设置 true 来禁用该功能。 | false                                                        |
| edit          | String          | 单元格开启编辑的方式（默认不开启），支持的值有： *text*（单行输入框） *textarea*（多行输入框）注：v2.7.0 新增 | -                                                            |
| event         | String          | 自定义单元格点击事件名，以便在 [tool](https://layui.dev/2.7/docs/modules/table.html#ontool) 事件中完成对该单元格的业务处理 | -                                                            |
| style         | String          | 自定义单元格样式。即传入任意的 CSS 字符                      |                                                              |
| align         | String          | 单元格排列方式。可选值有：*left*、*center*、*right*          | left                                                         |
| colspan       | Number          | 单元格所占列数。一般用于多级表头                             | 1                                                            |
| rowspan       | Number          | 单元格所占行数。一般用于多级表头                             | 1                                                            |
| templet       | String/Function | 自定义列模板，模板遵循 [laytpl](https://layui.dev/2.7/docs/modules/laytpl.html) 语法。这是一个非常实用的功能，你可借助它实现逻辑处理，以及将原始数据转化成其它格式，如时间戳转化为日期字符等 | [详见自定义模板](https://layui.dev/2.7/docs/modules/table.html#templet) |
| exportTemplet | String/Function | 专用于表格导出时的内容输出。 当上述 templet 较复杂时建议使用，如下以模板存在 select 元素为例：`</>exportTemplet: function(d, obj){  //当前 td  var td = obj.td(this.field);  //返回 select 选中值  return td.find('select').val();}`表格导出时，将优先以对应表头的 exportTemplet 的返回内容为导出结果。 注：v2.6.9 新增 | 用法同 templet                                               |
| toolbar       | String/Function | 绑定行工具模板。用法同 templet 参数完全相同（因为历史兼容问题所以保留）。 可在每行对应的列中出现一些自定义的操作性按钮 | [-](https://layui.dev/2.7/docs/modules/table.html#onrowtool) |

## 2. 头部工具栏



* html内容



```html
 <!-- 头工具栏 -->
<script type="text/html" id="toolbarDemo">
                      <div class="layui-btn-container">
                        <button class="layui-btn layui-btn-sm" lay-event="getCheckData" style="float: left">获取选中行数据</button>
                        <input type="text" name="username"  placeholder="请输入用户名" class="layui-input" style="width: 200px;float: left" id="searchinp">
                        <button class="layui-btn layui-btn-sm" style="float: left; margin-left: 5px" id="searchbtn">搜索</button>
                        <div class="layui-input-inline">
                            <select name="sex" lay-verify="required" lay-search="" lay-filter="sex">
                                <option value="">请选择</option>
                                <option value="男">男</option>
                                <option value="女">女</option>
                            </select>
                        </div>


                      </div>
</script>
```

* 开启头部工具栏

```js
  // 创建渲染实例
  table.render({
    elem: '#test1' //id值
    ,url: "{% url 'get_user' %}" // 对接后端服务的api
    ,title: 'user表'
    ,toolbar: '#toolbarDemo' //绑定id
    ,defaultToolbar: ['filter', 'print', 'exports']
    })
```

* 监听头部事件

```js
 //头工具栏事件
  table.on('toolbar(test)', function(obj){  //lay-filter值
    var checkStatus = table.checkStatus('USER'); //获取选中行状态
    switch(obj.event){
      case 'getCheckData':
        var data = checkStatus.data;  //获取选中行数据 数组,拿到行数据就可以ajax请求后端操作
        console.log(data)
        if (data != ''){
            layer.alert(JSON.stringify(data));
        }

      break;
    };
  });

```

* 监听行工具

```html
					<!-- 行工具栏 -->
                    <script type="text/html" id="toolDemo">
                      <div class="layui-btn-container">
                        <button class="layui-btn  layui-btn-normal" lay-event="edit">编辑</button>
                        <button class="layui-btn  layui-btn-danger" lay-event="del">删除</button>
                      </div>
                    </script>
```
