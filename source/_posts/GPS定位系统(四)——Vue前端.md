# 前言

GPS系列——Vue前端，[github项目地址](https://github.com/fly7632785/myadmin/tree/gps)

前面已经学习了Android、Java端的代码实现，现在开始介绍网站前端vue的管理框架。

文中也会有大量代码，对于admin管理框架，我是模仿[iview-amin](https://github.com/iview/iview-admin)，然后新建一个项目，手敲下来的，只取了自己所需的模块，目的就是为了练手，期间也遇到了很多问题，建议大家也可以自己模仿者手敲一遍。也可以使用[elementUI](https://element.eleme.cn/#/)，这个框架整体而言比iview更好一些。



### GPS定位系统系列

>[GPS定位系统(一)——介绍](https://www.jianshu.com/p/1aa130ffbeaf)
>
>[GPS定位系统(二)——Android端](https://www.jianshu.com/p/e463b1ea6b7d)
>
>[GPS定位系统(三)——Java后端](https://www.jianshu.com/p/2e51318c56e0)
>
>[GPS定位系统(四)——Vue前端](https://www.jianshu.com/p/9df016cd65fa)
>
>[GPS定位系统(五)——Docker](https://www.jianshu.com/p/75ba5bc90988)
>
>- [Docker nginx 二级域名无端口访问多个web项目](https://www.jianshu.com/p/3378d2eacb3d)
>- [Docker nginx https二级域名无端口访问多个web项目](https://www.jianshu.com/p/3e25b24d7247)
>- [持续部署——Travis+Docker+阿里云容器镜像](https://www.jianshu.com/p/ce648e120727)



### 收获

学习完这篇文章你将收获：

>- Vue + Vue-cli + iview + axios + vue-router + vuex 的实践
>- 高德地图 js api的使用
>- axios restful接口的异常处理封装
>- 上传头像
>- modal弹框编辑个人信息template
>- admin管理框架



[TOC]



# 正题

## 一、admin框架介绍

![主页](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702143344401.png)

框架搭建了整体架构，单页面的web应用。**通用缩放式菜单栏，选项卡式管理网页、面包屑导航、bug日志管理、全屏等功能。**

![image-20200710144618941](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200710144618941.png)

框架使用了通用热门的 VUE一套框架，包括[Vue](https://cn.vuejs.org/v2/guide/) + [Vue-cli](https://cli.vuejs.org/zh/) + [vuex](https://vuex.vuejs.org/zh/) + [vue-router](https://router.vuejs.org/zh/)+[iview](https://www.iviewui.com/docs/introduce)，具体还请参见源码。

单页面大致结构

```js
<template>
  <Layout style="height: 100%" class="main">
    <Sider ref="sider" class="sider" hide-trigger collapsible :collapsed-width="78" v-model="isCollapsed">
      <div class="logo-con">
        <img v-show="!isCollapsed" :src="maxLogo" class="max-logo" key="max-logo"/>
        <img v-show="isCollapsed" :src="minLogo" class="min-logo" key="min-logo"/>
      </div>
      <!--      展开状态-->
      <Menu class="open-menu" ref="menu" :active-name="$route.name" :open-names="openedNames" theme="dark" width="auto"
            v-show="!isCollapsed"
            @on-select="turnToPage">
        <template v-for="item in menuList">
          <!--          有children且只有1个-->
          <template v-if="item.children && item.children.length===1">
            <MenuItem :name='item.children[0].name'>
              <Icon :type="item.children[0].meta.icon"></Icon>
              <span>{{showTitle(item.children[0])}}</span>
            </MenuItem>
          </template>
          <template v-else>
            <!--            有children 大于1个嵌套-->
            <template v-if="item.children && item.children.length>1">
              <Submenu :name='item.name'>
                <template slot="title">
                  <Icon :type="item.meta.icon || ''"/>
                  <Span>{{showTitle(item) }}</Span>
                </template>
                <template v-for="subitem in item.children">
                  <MenuItem :name="subitem.name">
                    <Icon :type="subitem.meta.icon"></Icon>
                    <Span>{{showTitle(subitem)}}</Span>
                  </MenuItem>
                </template>
              </Submenu>
            </template>
            <!--            没有children-->
            <template v-else>
              <MenuItem :name='item.name'>
                <Icon :type="item.meta.icon"></Icon>
                <Span>{{showTitle(item)}}</Span>
              </MenuItem>
            </template>

          </template>

        </template>
      </Menu>
      <!--      收缩状态-->
      <div v-show="isCollapsed" class="close-menu">
        <template v-for="item in menuList">
          <template v-if="item.children && item.children.length>0">
            <Dropdown placement="right-start" @on-click="turnToPage" class='dropdown'>
              <a type="text" class="drop-menu-a">
                <Icon :type="item.meta.icon"></Icon>
              </a>
              <template v-for="subitem in item.children">
                <DropdownMenu slot="list">
                  <DropdownItem :name="subitem.name">
                    <a type="text" class="drop-item-a">
                      <Icon :type="subitem.meta.icon"></Icon>
                      <span>{{showTitle(subitem)}}</span>
                    </a>
                  </DropdownItem>
                </DropdownMenu>
              </template>
            </Dropdown>
          </template>
          <template v-else>
            <Tooltip transfer placement="right" :content="showTitle(item)">
              <a @click="turnToPage(item.name)" type="text" class="drop-menu-a">
                <Icon :type="item.meta.icon"></Icon>
              </a>
            </Tooltip>
          </template>
        </template>
      </div>
    </Sider>
    <layout>
      <Header class="header" :style="{padding:0}">

        <Icon @click.native="collapsedSider" :class="rotateIcon" :style="{margin:'0 20px'}" type='md-menu'
              size="24"></Icon>
        <custom-bread-crumb show-icon style="margin-left: 30px;" :list="breadCrumbList"></custom-bread-crumb>
        <div class="header-right">
          <user :message-unread-count="0" :user-avatar="userAvatar" :user-name="userName"/>
          <error-store v-if="$config.plugin['error-store'] && $config.plugin['error-store'].showInHeader"
                       :has-read="hasReadErrorPage" :count="errorCount"></error-store>
          <fullscreen v-model="isFullscreen" style="margin-right: 10px;"/>
        </div>
      </Header>
      <Content class="main-content-con">
        <Layout class="main-layout-con">
          <div class="tag-nav-wrapper">
            <tags-nav :value="$route" @input="handleClick" :list="tagNavList" @on-close="handleCloseTag"></tags-nav>
          </div>
          <Content class="content-wrapper">
            <keep-alive>
              <router-view/>
            </keep-alive>
          </Content>
        </Layout>
      </Content>
      <!--      <Footer class="footer">Footer</Footer>-->
    </layout>
  </Layout>
</template>
```

对于缩放菜单的功能较为复杂，可以细品一下，能收获许多。



## 二、axios封装

我们的java后台的接口统一数据为restful结构的

```
{code:xxx,msg:xxx,data:xxx}
```

对于axios而言封装上面要注意其返回的respon的结构，以及异常response的结构的处理。

这里先放**整体axios封装**代码：

**axios.js**

```js
import axios from 'axios'
import store from '@/store'
import {Message} from 'iview'

// import { Spin } from 'iview'
const addErrorLog = errorInfo => {
  const {statusText, status, request: {responseURL}} = errorInfo
  let info = {
    type: 'ajax',
    code: status,
    mes: statusText,
    url: responseURL
  }
  console.log("addErr:" + JSON.stringify(info))
  if (!responseURL.includes('save_error_logger')) store.dispatch('addErrorLog', info)
}

class HttpRequest {
  constructor(baseUrl = baseURL) {
    this.baseUrl = baseUrl
    this.queue = {}
  }

  getInsideConfig() {
    const config = {
      baseURL: this.baseUrl,
      // headers: {
      //   'Content-Type': "application/json;charset=utf-8"
      // }
    }
    return config
  }

  destroy(url) {
    delete this.queue[url]
    if (!Object.keys(this.queue).length) {
      // Spin.hide()
    }
  }


  interceptors(instance, url) {
    // 请求拦截
    instance.interceptors.request.use(config => {
      // 添加全局的loading...
      if (!Object.keys(this.queue).length) {
        // Spin.show() // 不建议开启，因为界面不友好
      }
      this.queue[url] = true
      return config
    }, error => {
      return Promise.reject(error)
    })
    // 响应拦截
    instance.interceptors.response.use(res => {
      console.log("res:" + JSON.stringify(res))
      this.destroy(url)
      const {data: {code, data, msg}, config} = res
      if (code == 200) {
        return data;
      } else {
        this.dealErr(code, msg)
        let errorInfo = {
          statusText: msg,
          status: code,
          request: {responseURL: config.url}
        }
        addErrorLog(errorInfo)
        return Promise.reject(res.data)
      }
    }, error => {
      console.log("error:" + JSON.stringify(error))
      this.destroy(url)
      let errorInfo = error.response
      if (!typeof(errorInfo) === undefined && !errorInfo) {
        const {request: {statusText, status}, config} = JSON.parse(JSON.stringify(error))
        errorInfo = {
          statusText,
          status,
          request: {responseURL: config.url}
        }
        addErrorLog(errorInfo)
        const data = {code: status, msg: statusText}
        this.dealErr(data.code, data.msg)
      } else {
        Message.error('网络出现问题，请稍后再试')
      }
      return Promise.reject(error)
    })
  }


  dealErr(c, msg) {
    console.log("code:" + c)
    console.log("msg:" + msg)
    switch (c) {
      case 400:
        Message.error(msg)
        break;
      case 401:
        Message.error('登录过期，请重新登录')
        break;
      // 404请求不存在
      case 404:
        Message.error('网络请求不存在')
        break;
      // 其他错误，直接抛出错误提示
      default:
        Message.error("系统错误")
    }
  }

  request(options) {
    const instance = axios.create()
    options = Object.assign(this.getInsideConfig(), options)
    this.interceptors(instance, options.url)
    return instance(options)
  }


}

export default HttpRequest

```

**api.request.js:**

```js
import HttpRequest from '@/libs/axios'
import config from '@/config'
const baseUrl = config.baseUrl

const axios = new HttpRequest(baseUrl)
export default axios

```

**调用：**

```js
import axios from '@/libs/api.request'
import Qs from 'qs'

export const login = ({username, password}) => {
  const data = {
    username,
    password
  }
  return axios.request({
    url: 'login',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    data: Qs.stringify(data),
    method: 'post'
  })
}
```

**注意**：如果要使用**form表单**的形式，需要做**转化**，这里可以简单方便的使用`Qs`库来直接`stringify`，也别忘了设置headers的`'Content-Type': 'application/x-www-form-urlencoded'`，因为axios默认的是json格式。

### 1、interceptors的response编写

```js
res => {
      console.log("res:" + JSON.stringify(res))
      this.destroy(url)
      const {data: {code, data, msg}, config} = res
      if (code == 200) {
        return data;
      } else {
        this.dealErr(code, msg)
        let errorInfo = {
          statusText: msg,
          status: code,
          request: {responseURL: config.url}
        }
        //添加到日志
        addErrorLog(errorInfo)
        return Promise.reject(res.data)
      }
    }
```

```json
{
    "data":{
        "code":200,
        "data":{
            xxxx
        },
        "msg":"请求成功"
    },
    "status":200,
    "statusText":"",
    "headers":{
        "content-length":"487",
        "content-type":"application/json;charset=UTF-8"
    },
    "config":{
        "url":"http://127.0.0.1:9090/get_info",
        "method":"post",
        "headers":{
            "Accept":"application/json, text/plain, */*",
            "token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJsb2dpbk5hbWUiOiJrayIsImV4cCI6MTU5Njk1Nzg1MSwidXNlcklkIjoiMTMifQ.ChaBg4n5KKsF7ISj8uzHV0eh_JKadoVIBtNG4oUtp8U"
        },
        "baseURL":"http://127.0.0.1:9090/",
        "transformRequest":[
            null
        ],
        "transformResponse":[
            null
        ],
        "timeout":0,
        "xsrfCookieName":"XSRF-TOKEN",
        "xsrfHeaderName":"X-XSRF-TOKEN",
        "maxContentLength":-1
    },
    "request":{

    }
}
```

注意，axios接口请求的response的数据结构如上。

可以看到获取数据需要`res.data.data`，我们这里用解构 `const {data: {code, data, msg}, config} = res` 一下，code==200(其实是res.data.code)的时候返回data（其实就是返回res.data.data）

### 2、error处理

这里的error可以分为3类：

- 服务器端的业务的http error
- 前端的http error
- 前端**非**http error

```js
dealErr(c, msg) {
    console.log("code:" + c)
    console.log("msg:" + msg)
    switch (c) {
      case 400:
        Message.error(msg)
        break;
      case 401:
        Message.error('登录过期，请重新登录')
        break;
      // 404请求不存在
      case 404:
        Message.error('网络请求不存在')
        break;
      // 其他错误，直接抛出错误提示
      default:
        Message.error("系统错误")
    }
  }
```

这里封装一个方法，用于处理**服务器端的业务的http error**和**前端的https error**，因为他们结构都是相同的。

```js
error => {
      console.log("error:" + JSON.stringify(error))
      this.destroy(url)
      let errorInfo = error.response
      if (!typeof(errorInfo) === undefined && !errorInfo) {
        const {request: {statusText, status}, config} = JSON.parse(JSON.stringify(error))
        errorInfo = {
          statusText,
          status,
          request: {responseURL: config.url}
        }
        addErrorLog(errorInfo)
        const data = {code: status, msg: statusText}
        this.dealErr(data.code, data.msg)
      } else {
        Message.error('网络出现问题，请稍后再试')
      }
      return Promise.reject(error)
    }
```

但是还有一种error也要处理，这里判断如果**error.response不为空**，则为**http类型的error**，使用dealErr，如果为空，则说明是**非http类型的err**，直接toast **网络出现问题，请稍后再试**（比如，前端跨域报错，请求不符合规范等错误，就是非http类型的err）

结构如下

```json
{
    "message":"Network Error",
    "name":"Error",
    "stack":"createError handleError",
    "config":{
        "url":"http://127.0.0.1:9090/login",
        "method":"post",
        "data":"username=kk&password=kk",
        "headers":{
            "Accept":"application/json, text/plain, */*",
            "Content-Type":"application/x-www-form-urlencoded"
        },
        "baseURL":"http://127.0.0.1:9090/",
        "transformRequest":[
            null
        ],
        "transformResponse":[
            null
        ],
        "timeout":0,
        "xsrfCookieName":"XSRF-TOKEN",
        "xsrfHeaderName":"X-XSRF-TOKEN",
        "maxContentLength":-1
    }
}
```



## 三、高德地图相关功能

### 1、引入sdk使用

**1）**在public文件夹下的index.html的<head>中加入 `<script type="text/javascript" src="https://webapi.amap.com/maps?v=1.4.15&key=你的key"></script>`

**注意**这里需要在body前面，不然有时候地图加载不出来

vue.config.js配置文件中加入

```js
module.exports = {
  configureWebpack: {
    externals: {
      'AMap': 'AMap',
      'AMapUI': 'AMapUI'
    },
  },
}
```

**2）**vue文件的template中加入``<div id="map"></div>`  ` 

**注意**map需要设置宽高

```
#map{
  width: 100%;
  height: 100%;
}
```

文件中引入mapUI

```
<script src="//webapi.amap.com/ui/1.1/main.js"></script>
<script>
 import AMap from 'AMap'
```

**3）**使用

```js
 const map = new AMap.Map('map', {
          resizeEnable: true,
          zoom: 11
        })
        this.map = map
        map.plugin(['AMap.ToolBar', 'AMap.MapType'], function () {
          map.addControl(new AMap.ToolBar())
          map.addControl(new AMap.MapType({showTraffic: false, showRoad: false}))
        })
```

**注意：**如果同一页面使用多个地图，需要div的id设置为不同，比如，map1、map2等，Map()构造也要传入相应的map id

不出意外，地图就能加载出来了。

### 2、实时定位

![实时定位](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702143431034.png)

**实时定位功能主要逻辑为：**

**1、从接口获取所有人的最新的定位信息，然后生成marker标注在地图上**

**2、并且给每个标注都注入个人信息，如头像、名字、时间等信息**

**3、可以下拉选择用户，定位到地图中心，且打开infoWindow展示个人信息**

**4、由于接口是没有存具体地址的，需要利用经纬度坐标，转为具体地址展示在infowindow上**

详情，请参看源码。

获取接口数据，添加markers和dropdown

```js
 allNowGps() {
        this.getAllNowGps().then(res => {
          this.data = res;
          this.addMarkers(res)
          this.addDropDown(res)
          console.log("res", JSON.stringify(res))
        })
      },
 addMarkers(data) {
        this.infoWindow = new AMap.InfoWindow({offset: new AMap.Pixel(0, -70)});
        data.forEach((item, index) => {
          var marker = new AMap.Marker({
            position: [item.lng, item.lat],
            icon: personLogo,
            offset: new AMap.Pixel(-15, -66),
            map: this.map,
            extData: item,
          });
          var info = this.getContentByItem(item);
          marker.content = info.join("<br/>") //使用默认信息窗体框样式，显示信息内容
          marker.on('mouseover', this.markerHover);
          // marker.emit('mouseover', {target: marker});
          // 真正的content的使用，是在hover打开infowindow的时候，传给indowindow进行展示的
        })
        this.map.setFitView();
      },
addDropDown(res) {
        if (res && res.length > 0) {
          res.forEach(item => {
            //默认获取自己的
            //注意从cookie里面拿出来默认是string
            if (item.uid == this.$store.state.user.userId) {
              this.currentUser = item
              //设为中心 显示信息
              this.map.setCenter([item.lng, item.lat])
            }
          })
        }
      },
```

hover之后展示infowindow

```js
 markerHover(e) {
        var _this = this
        //转换经纬度为具体地址
        this.geocoder.getAddress(e.target.getPosition(), function (status, result) {
          if (status === 'complete' && result.info === 'OK') {
            var address = result.regeocode.formattedAddress;
            console.dir(address);
            const content = e.target.content + '<br/>地址：' + address;
            const marker = e.target
            const item = e.target.getExtData()
            console.log('extData', e.target.getExtData());
            _this.infoWindow.setContent(content)
            _this.infoWindow.open(_this.map, e.target.getPosition());
          }
        });
      },
```

这里**e.target**其实就是marker，然后marker的**extData**可以装载 item的数据，这里也取出来展示

注意，这里使用了**geocoder**来把经纬度坐标转为具体地址

### 3、历史轨迹

![历史轨迹](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702143808473.png)

**历史轨迹功能主要逻辑为：**

**1、默认获取当前用户的历史轨迹数据，可以通过日期筛选，并生成轨迹和marker**

**2、获取所有的用户，生成选择下拉列表，选择下拉，可以获取对应用户的历史轨迹数据**

**3、利用高德地图的绘制轨迹**

**4、开始marker沿着轨迹移动，模拟移动行为**

```js
 allUsers() {
        this.handleGetAllUsers().then(res => {
          console.log("users", JSON.stringify(res))
          let users = [];
          users = res;
          this.users = users
          if (res && users.length > 0) {
            users.forEach(item => {
              //默认获取自己的
              //注意从cookie里面拿出来默认是string   == 就可以比较
              if (item.uid == this.$store.state.user.userId) {
                this.currentUser = item
                this.selectGpsHis(item.uid)
              }
            })
          }
        })
      },
```

获取所有用户

```js
selectGpsHis(uid, from, to) {
        this.getGpsHis({uid, from, to}).then(data => {
          console.log("getGpsHis", JSON.stringify(data))
          this.showGpsHis(data)
        })
      },
```

获取对应用户的历史轨迹数据

```js
showGpsHis(data) {
        this.map.clearMap()
        this.followPath = []

        data.forEach((item, index) => {
          const gps = [item.lng, item.lat]
          this.followPath.push(gps)
        })
        //重组数据为 [[lng,lat],[lng2,lat2]]

        if (this.followPath.length === 0) {
          Message.warning('无历史数据')
          return
        }

        this.marker = new AMap.Marker({
          map: this.map,
          position: this.followPath[0],
          icon: personLogo,
          offset: new AMap.Pixel(-15, -66),
        });

        // 绘制轨迹
        var polyline = new AMap.Polyline({
          map: this.map,
          path: this.followPath,
          showDir: true,
          strokeColor: "#28F",  //线颜色
          // strokeOpacity: 1,     //线透明度
          strokeWeight: 6,      //线宽
          // strokeStyle: "solid"  //线样式
        });

        var passedPolyline = new AMap.Polyline({
          map: this.map,
          // path: this.followPath,
          strokeColor: "#AF5",  //线颜色
          // strokeOpacity: 1,     //线透明度
          strokeWeight: 6,      //线宽
          // strokeStyle: "solid"  //线样式
        });

        this.marker.on('moving', function (e) {
          passedPolyline.setPath(e.passedPath);
        });
        this.map.setFitView()
      },
```

绘制轨迹并且生成marker

```js
 startAnimation() {
        this.marker.moveAlong(this.followPath, 5000);
      },
stopAnimation() {
          this.marker.stopMove()
        },
```

开始轨迹、停止轨迹，轨迹速度是按照地图的**每小时多少千米**的速度来设置的。

**注意**：拿到的数据，需要**重组**成 轨迹所需的数据结构。

## 四、modal弹框template自定义

![用户管理](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702143910659.png)

对于用户管理，我们有创建和编辑用户资料，两个格式相同，可以复用。这里自定义的modal模板，可以参考一下。

代码较长：

```js
<template>
  <Modal :value="isShow" :title="title" @on-visible-change="handleVisible">
    <Upload
      ref="upload"
      :show-upload-list="false"
      :on-success="handleUploadSuccess"
      :format="['jpg','jpeg','png']"
      :max-size="2048"
      :on-format-error="handleUploadFormatError"
      :on-exceeded-size="handleUploadMaxSize"
      :headers="header"
      type="drag"
      :action="uploadUrl"
      style="display: inline-block;width:50px;height:50px;margin-bottom: 50px">
      <Avatar :src='user.avatar' style="width:50px;height: 50px"/>
    </Upload>
    <Form ref="user" :model="user" :rules="ruleValidate">
      <FormItem label="用户名" prop="username">
        <Input v-model="user.username"/>
      </FormItem>
      <FormItem label="密码" prop="password">
        <Input v-model="user.password"/>
      </FormItem>
      <FormItem label="姓名">
        <Input v-model="user.name"/>
      </FormItem>
      <FormItem label="手机" prop="mobile">
        <Input v-model="user.mobile"/>
      </FormItem>
    </Form>
    <div slot="footer">
      <Button size="large" @click="handleCancel">取消</Button>
      <Button type="primary" size="large" @click="handleConfirm('user')">确定</Button>
    </div>
  </Modal>
</template>
<script>
  export default {
    name: 'edit-user',
    props: {
      //姓名、头像、手机、用户名、密码、
      user: {
        uid: '',
        avatar:'',
        name: '',
        username: '',
        password:null,
        mobile: '',
      },
      //是否显示弹框
      isShow: false,
      //上传所需的token
      header: {},
      //上传地址
      uploadUrl: {},
      isEdit: {
        type: Boolean
      },
    },
    computed: {
      title() {
        return this.isEdit === true ? "修改用户信息" : "新建用户信息"
      },
      ruleValidate() {
        console.log('ruleValidate', this.isEdit)
        const rule = this.isEdit === true ? this.editRuleValidate : this.createRuleValidate
        console.log('ruleValidate', rule)
        return rule
      }
    },
    watch:{
      isShow(val){
        console.log('isShow',val)
      }
    },
    data() {
      return {
        editRuleValidate: {
          username: [
            {required: true, message: 'The name cannot be empty', trigger: 'blur'}
          ],
          mobile:[
            { required: false, message: "请输入手机号码", trigger: "blur" },
            { pattern: /^1[3456789]\d{9}$/, message: "手机号码格式不正确", trigger: "blur"}
          ]
        },
        createRuleValidate: {
          username: [
            {required: true, message: 'The name cannot be empty', trigger: 'blur'}
          ],
          password: [
            {required: true, message: 'The password cannot be empty', trigger: 'blur'}
          ],
          mobile:[
            { required: false, message: "请输入手机号码", trigger: "blur" },
            { pattern: /^1[3456789]\d{9}$/, message: "手机号码格式不正确", trigger: "blur"}
          ]
        }
      }
    },
    methods: {
      handleUploadSuccess(res, file) {
        console.log(file.response)
        this.user.avatar = file.response
      },
      handleUploadFormatError(file) {
        this.$Notice.warning({
          title: 'The file format is incorrect',
          desc: 'File format of ' + file.name + ' is incorrect, please select jpg or png.'
        });
      },
      handleUploadMaxSize(file) {
        this.$Notice.warning({
          title: 'Exceeding file size limit',
          desc: 'File  ' + file.name + ' is too large, no more than 2M.'
        });
      },
      //处理确定
      handleConfirm(name) {
        this.$refs[name].validate((valid) => {
          console.log('handleConfirm validate',valid)
          if (valid) {
            //发送ok事件
            this.$emit('ok', {user: this.user, isEdit: this.isEdit})
            //关闭弹框
            this.$emit('visible', false)
          }
        })
      },
      handleCancel() {
        //关闭弹框
        this.$emit('visible', false)
      },
      handleVisible(visible) {
        //每次都清空验证信息 因为编辑和创建不一样
        this.$refs['user'].resetFields();
        //发送事件给父组件 修改自己的visible状态（注意这里 不能用v-model数据绑定 子组件不能修改父组件传来的prop的对象状态）
        this.$emit('visible', visible)
      }
    },
  };
</script>

```

使用：

```js
<div v-if="isShowEdit">
      <EditUser :user="newUser"
                :isEdit="isEdit"
                @visible="this.handleEditVisible"
								:is-show="true"
                :header="{'token': this.$store.state.user.token}"
                :upload-url="this.$config.baseUrl + 'upload'"
                @ok="handleEditOk"
      ></EditUser>
    </div>
```

通过**isEdit**去判别是编辑还是新建

**这里有一点比较重要：**

**isShowEdit**是用于我们动态去展示和隐藏modal弹框，使用的是v-if。网上很多人，是使用modal的v-model或者value来控制modal的显示和隐藏的，原本我也是那样做的，但是后来发现，那样做非常不稳定和可靠，有时候弹框弹出来，其双向绑定的**is-show**，并未能和modal的状态统一。所以，最终使用的是div+v-if的方式来控制。



**还有就是，关于父组件和子组件之间传值的问题：**

父给子，一般是通过props来接收，并且，我们希望的是**单向**的，父可以改变控制子，但是，子不能改变去控制父，不然会报错。而需要使用，**子发事件回调给父，来改变父的状态的方式**来实现。

```js
//处理确定
      handleConfirm(name) {
        this.$refs[name].validate((valid) => {
          console.log('handleConfirm validate',valid)
          if (valid) {
            //发送ok事件
            this.$emit('ok', {user: this.user, isEdit: this.isEdit})
            //关闭弹框
            this.$emit('visible', false)
          }
        })
      },
      
 handleVisible(visible) {
        //每次都清空验证信息 因为编辑和创建不一样
        this.$refs['user'].resetFields();
        //发送事件给父组件 修改自己的visible状态（注意这里 不能用v-model数据绑定 子组件不能修改父组件传来的prop的对象状态）
        this.$emit('visible', visible)
      }
```

`this.$emit('visible', false)` 使用这个来改变其父的`isShowEdit`的值，从而隐藏或者显示自身modal。



### 上传头像

```js
<Upload
      ref="upload"
      :show-upload-list="false"
      :on-success="handleUploadSuccess"
      :format="['jpg','jpeg','png']"
      :max-size="2048"
      :on-format-error="handleUploadFormatError"
      :on-exceeded-size="handleUploadMaxSize"
      :headers="header"
      type="drag"
      :action="uploadUrl"
      style="display: inline-block;width:50px;height:50px;margin-bottom: 50px">
      <Avatar :src='user.avatar' style="width:50px;height: 50px"/>
    </Upload>
```

上传头像注意下，如果我们上传图标需要header，比如传token的话，需要把**header**作为参数传进来。



## 五、登录验证

对于登录验证，我们这边是使用vue-router的beforeEach统一处理页面的跳转来实现的。

```js
const router = new Router({
  routes: routers,
  mode: 'history',
})
const turnTo = (to, access, next) => {
  // if (canTurnTo(to.name, access, routes)) next() // 有权限，可访问
  // else next({replace: true, name: 'error_401'}) // 无权限，重定向到401页面
  next()
}
router.beforeEach((to, from, next) => {
  iView.LoadingBar.start()
  const token = getToken()
  if (!token && to.name !== LOGIN_PAGE_NAME) {
    // 未登录且要跳转的页面不是登录页
    next({
      name: LOGIN_PAGE_NAME // 跳转到登录页
    })
  } else if (!token && to.name === LOGIN_PAGE_NAME) {
    // 未登陆且要跳转的页面是登录页
    next() // 跳转
  } else if (token && to.name === LOGIN_PAGE_NAME) {
    // 已登录且要跳转的页面是登录页
    next({
      name: homeName // 跳转到homeName页
    })
  } else {
    //这由于暂时没有权限系统 直接 跳转即可
    // if (store.state.user.hasGetInfo) {
      turnTo(to, store.state.user.access, next)
    // } else {
    //   store.dispatch('getUserInfo').then(user => {
    //     // 拉取用户信息，通过用户权限和跳转的页面的name来判断是否有权限访问;access必须是一个数组，如：['super_admin'] ['super_admin', 'admin']
    //     turnTo(to, user.access, next)
    //   }).catch(() => {
    //     setToken('')
    //     next({
    //       name: 'login'
    //     })
    //   })
    // }
  }
})

router.afterEach(to => {
  //设置标题
  setTitle(to, router.app)
	//隐藏进度条
  iView.LoadingBar.finish()
  window.scrollTo(0, 0)
})
```





# 总结

源码很多，所以其实很多东西都在源码里面了，可能内容篇幅较长，很少有人能够完整看完，但是，写在这里只为某些时候可能遇到类似问题，有一个借鉴参考的地方即可。就如同，我自己手敲admin框架的时候，很多时候iview-amin就是我的一个可以借鉴和参考的项目，遇到有不会的或者没有思路的，就可以参考借鉴一下，这样会好很多。

整个系列，前端、移动端、后端，都有了，打通了，接下来就是要学习一下，怎么打包，部署到服务器那些东西了。

服务器呢，我打算再使用docker，学习一番docker+nginx+mysql等实现前端和后端的线上部署，具体请参看

[GPS定位系统(五)——Docker](https://www.jianshu.com/p/75ba5bc90988)





# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts