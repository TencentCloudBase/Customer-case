11月的某天，本科死党突然在群里分享了微信小程序马拉松比赛的消息，自己脑子一热居然答应了...在冷静之余，作为一个Java和C++的后端工程师（对，我不会nodejs...），自己脑子里首先蹦出了一个todolist:

>- 环境：腾讯的云裸机或者Docker环境，0.5day
- 工程开发脚手架：spring boot，鉴于好久没搞java了， 1day
- 微信开发者文档阅读：应该会要有一套鉴权机制，之前没有研究过，1day

可是有拖延症的我在邻近比赛开始前2天（周四晚）依然还什么都没有准备，正当怀着赴死（被队友砍死）心态刷着比赛群里的信息时，腾讯云同学的两页PPT引起了我的注意。

![](https://puui.qpic.cn/vupload/0/20190723_1563874819070_ngvrpvfg9x.jpeg/0)

第一页PPT是否意味着，这次比赛会有一个FaaS服务，可以 **让我不关注于系统运维以及各种web框架的路由策略？**

![](https://puui.qpic.cn/vupload/0/20190723_1563874865870_zpt04cvb0j.jpeg/0)

第二页PPT是否意味着，这个FaaS服务还可以 **帮助我处理一系列繁琐的用户鉴权服务?**

总之，求生欲让我点进了 **小程序·云开发文档** ，读完发现————**真！牛！逼！** 看来我有望 **依托云开发在24小时内完成nodejs语言的入门及后端服务的搭建！**

果然，在接下来的时间，凭借着云开发，最终我们组 **1** 个C++研发、**1** 个算法工程师和 **1 个前端大腿** 在24小时内完成了最小可行的小程序，并在最后的评比中获得三等奖。下面简单分享一下这次黑客马拉松中我们使用云开发的体验，更细节的点可以移步云开发文档。

## 实战

本节主要涉及马拉松项目的由来，简单的系统架构图。然后，列举了几个典型依托云开发的功能实现。

### 一、功能描述

一番头脑风暴后，我们组最终决定要继续zuo死，搞一个略复杂的 **英文配音小程序** 。项目背景源于在互助学习群中，会有学生发送语音给老师希望得到点评的需求，但是现有微信聊天机制使得这样的模式很不容易管理。因此我们希望通过一个小程序进来改变这种现状，此外，这个小程序最好还能增加一些趣味性，在简单的讨论后，我们的系统流程是这样的。

![](https://puui.qpic.cn/vupload/0/20190723_1563875052600_wtwbx530rw.jpeg/0)

整个闭环中有老师和学生两种角色。当然在马拉松中，为了快速演示，我们目前是hard code 一名用户为老师，其他角色均为学生。老师可以发起课程，并将课程分享到群中，学生通过点击群中分享进入到配音详情页面，在完成配音后，老师会受到评价提醒的模板消息，并进行配音评价，在老师完成评价后，学生会受到反馈提醒的模板消息，去查看评价，从而完成了整个交互。

### 二、系统总体实现

如功能描述中所述，整个小程序的主要技术难点在交互流程状态的维护和音频合成能力。对于交互流程的状态维护，我们使用云开发完快速解决。不过，由于目前云开发不支持在云环境部署一些自定义程序，我们依然通过IaaS虚拟机部署的方式实现了视频拼接服务。总体来看，系统总体模块如下图所示。

![](https://puui.qpic.cn/vupload/0/20190723_1563875102662_9npoy85ocaq.png/0)

数据视图的设计上，我们主要定义了课程、用户formid、用户信息、作品以及课程素材五类数据，依托于云开发的数据库，在数据的设计上并没有费太多功夫。

![](https://puui.qpic.cn/vupload/0/20190723_1563875136929_7j79bkfjdng.jpeg/0)

### 三、选择课程&开启课程

选择课程&开启课程是我们开发中比较典型的一环，主要由小程序端发起调用，在云函数中对数据库进行操作，最终返回结果。

在小程序端，为了方便调用，都是通过工厂模式对调用云端函数进行了简单封装，代码片段如下：

```javascript
/**
 * 云函数调用
 * @param name
 * @param data
 * @returns {*}
 */
function callCloudFunctionFactory (name,data) {   
    let OPENID = wx.getStorageSync('__open_id');
    console.log('OPENID:'+OPENID);
    return callFunction({
        name: name,
        data: Object.assign({
            openId: OPENID
        },data)
    })
}
```

有了调用的封装，当老师点击创建课程的时候，就会调用云端`creat_class`的请求。

```javascript
cloundApi.createClass({item_detail:item}).then((res)=>{
    item['class_id'] = res.result;
    wx.setStorageSync('_selected_item', item);
    wx.navigateBack();
    wx.hideLoading();
}).catch((res)=>{
    util.showToast('创建课程失败');
    wx.hideLoading();
})
```

在云端，创建课程的逻辑也非常简单，云函数首先会去调用获取用户信息的云函数，得到用户的基本信息，并生成一个UUID作为课程标识，并将用户信息与课程关联起来，存储数据库中，代码片段如下。

```javascript
exports.main = async (event, context) => {  
  try {
    const res = await cloud.callFunction({
      // 要调用的云函数名称
      name: 'get_user_info',
      // 传递给云函数的参数
      data: {
        openId: event.openId
      }
    })
    event.profile = res.result
    let UUID = require('uuid')
    event.class_id = UUID.v1()
    console.log(event)
    let result = await db.collection('collection_class').add({
      data: event
    })
    if (result._id) {
      return event.class_id
    } else {
      return false
    }
  } catch(e) {
    console.error(e)
  }
}
```

### 四、音频处理

交互配音比较遗憾的地方在于，后端配音合成需要用到一款名为ffmeg的软件，而云函数并不支持我们对环境进行定制。考虑到开发时间有限，以及 **前端同学有较丰富的小程序端开发经验** ，此处我们的策略是快速基于腾讯云部署一个IaaS服务，并依托SpringBoot 实现ffmeg的http封装。此时IaaS端的SpringBoot服务仅仅是提供视频合成的能力，在合成了视频之后会返回视频的UUID，小程序端将这段uuid通过云函数存储数据库。在需要播放视频时，也是先通过云函数拿到uuid，再去IaaS端下载。

此处小程序端上传音频的代码片段为

```javascript
api.createMatchFilm({
    item_id: this.data.query.item_id,
    dub_list: this.data.recordedVoiceUploadedList,
    user_role: userRole
}).then((res)=>{
    this.bgVideoContext.pause();
    this.originVideoContext.pause();
    recordInnerAudioContext.stop();
    originInnerAudioContext.stop();
    clearInterval(this.playAllRecordVoiceLoop);
})
```

这里同样使用了类似于工厂模式的方式发起对IaaS端的调用。在IaaS端完成合成后，会返回一段UUID，小程序端将调用云函数存储课程信息，同时触发通知老师的动作。云函数端的代码片段为：

```javascript
const classInfo = await cloud.callFunction({
      // 要调用的云函数名称
      name: 'get_class_info',
      // 传递给云函数的参数
      data: {
        class_id: event.class_id,
      }
    })
    let teacher_id = classInfo.result.openId
    if (result._id){
      const pushTeacher = await cloud.callFunction({
        // 要调用的云函数名称
        name: 'push_msg_to_teacher',
        // 传递给云函数的参数
        data: {
          openId: teacher_id,
          work_id: result._id,
          detail: classInfo.result.item_detail.film_name,
          nick: event.profile.nick
        }
      })
      return {
          work_id:result._id
      }
    }else{
      return false
    }
```

这里，云函数首先会将信息存储入数据库，然后通过发送模板消息的方式通知老师。

### 五、模板消息推送

依托于云开发丰富的API，后端也能快速搭建模板消息推送的链路。模板消息推送主要需要完成小程序端搜集formid，云端推送模板消息两个部分。

首先是formid采集，小程序端会记录用户触发的formid，并写入云端函数库，云端将每个用户的formid以类似于队列的形式存储。

首先在创建用户时，同时增加一个用于存储用户formid的队列。

```javascript
let from_id_result = await db.collection('collection_form_id').add({
  data: {
    _id: event.openId
  }
})
```

然后在小程序端增加用户formid时将formid放入队列

```javascript
exports.main = async (event, context) => {
  const wxContext = cloud.getWXContext()
  var openId = wxContext.OPENID
  if (openId == undefined || event.formId == null) {
    return false
  }

  try {
    return await db.collection('collection_form_id').doc(openId).update({
      data: {
        tags: _.push([event.formId])
      }
    })
  } catch (e) {
    console.error(e)
    return false
  }
  return true
}
```

在需要发送模板消息时，将formid从队头取出，并调用微信发送模板消息接口，向用户发送模板消息。下面代码展示了云函数取出formid，生成模板消息的过程。

```javascript
exports.main = async (event, context) => {
  let wXMINIUser = new WXMINIUser({ appId, secret })
  let access_token = await wXMINIUser.getAccessToken()
  const wxContext = cloud.getWXContext()
  // to 学生
  const openId = event.openId
  const nick = event.nick
  const class_id = event.class_id

  const templateId = 'xxxxxx' // 小程序模板消息 template ID

  if (openId == undefined) {
    console.log("openId is undefined")
    return false
  }
  let tmp = await db.collection('collection_form_id').doc(openId).get()

  if (tmp.data.tags == undefined || tmp.data.tags.length == 0){
    console.log("tmp.data.tags used out")
    return false
  }

  let formId = tmp.data.tags[0]
  console.log("formeId is: ")
  console.log(formId)
  await db.collection('collection_form_id').doc(openId).update({
    data: {
      tags: _.shift()
    }
  })

  let wXMINIMessage = new WXMINIMessage({
    openId,
    formId,
    templateId
  })

  return await wXMINIMessage.sendMessage({
    access_token,
    data: {
      keyword1: {
        value: nick
      },
      keyword2: {
        value: '【点击查看老师对你本次口语细节的纠正和指导】'
      },
    },
    page: '/pages/home/index?class_id='+class_id // 点击模板消息后，跳转的页面
  })
}
```

### 云开发体验小结

总体来说，我认为一般业务开发会遇到以下三个痛点：

> - 前后端调用，处理与通信（网络、编码、鉴权）相关问题
- 与数据库联调、同样处理与连接、编码相关问题，并且要考虑数据库的运维
- 开发时的单元测试及前后端联调成本

其实，这些问题都是与业务无关的底层问题，如果没有熟悉的搭档或者成熟的脚手架，大量时间将被消耗在这样的胶水代码中。不过这次有了云开发，情况变得有些不一样。

### 一、如丝般顺滑的前后联调

正如实战中代码所示，有了云开发，后端就彻底从胶水代码中解放出来。因此，在和 **前端大腿** 定义好前后端交互接口后，我们和另一名队友就靠着云开发接口文档和w3c中nodejs入门开始了后端开发之旅，并在一个下午的功夫就完成了下图中函数的开发。

![](https://puui.qpic.cn/vupload/0/20190723_1563877550497_6f42umvh40v.jpeg/0)

### 二、轻量级易调试的NoSQL存储

除了需要完成和前端的交互，后端逻辑还需要做一个“状态的维护”，即实现和数据库的交互。虽然在像spring boot这样的框架中，已经提供了一整套便捷的orm解决方案，云开发在这方面更进一步，主要体现在一下两个方面：

1. 部署侧，屏蔽数据库运维细节，直接提供NoSQL能力以及直观的方便操作的dashboard
2. 调用侧，用户无需关注通信、连接池、编码等细节问题，直接通过封装好的API对数据库进行调用。（事实上，云开发还可以允许前端对数据库进行操作，但是需要调用者在操作时明确调用的读写权限）

### 三、提升开发效率的dashboard

除了封装底层胶水代码提升coding效率，云开发的dashboard也能提升开发时的调试效率。 首先是云函数的dashboard，一方面，支持函数粒度的单元测试，在写好函数后，后端同学可以快速定义运行时的入参，检查函数的调用结果。由于屏蔽了底层通信细节， **此时后端同学的测试用例和前端最终使用的入参是完全一致的** ，因此只要后端的单元测试覆盖了所有的case，前后联调就是一个 **100% Bug Free** 过程。此外，dashboard中还提供了简单的日志分析功能，方便快速定位问题以及调试。

![](https://puui.qpic.cn/vupload/0/20190723_1563877666450_jnav0e8quj.jpeg/0)

除此以外，不得不赞一个数据库dashboard，一方面添加记录的交互直观明了，另一方面对json数据导入导出的支持大大提升在开发时构造测试数据的效率。

![](https://puui.qpic.cn/vupload/0/20190723_1563877700968_wl2k8ai7hc8.jpeg/0)

除此外，还有类似于用户管理、存储管理、统计分析的dashboard，他们都提供了一些“顾名思义”的功能。因为是在比赛中，就也没有花太多的时间去仔细了解，不过给我的感觉是，这些功能都是小而美、稳定高效的。

### serverless，赋能敏捷开发

16年小程序刚出来的时候，自己就有过一次小程序的开发体验，当时就觉得这是一个简洁而高效的端上开发体验。

而现在的云开发，又刚好完成了服务侧的拼图，提供了对底层透明的开发体验。

现在看来，在小程序的生态下，开发的门槛出奇的低：可能仅仅是像我这样“入门”nodejs的程序员，或者哪怕是一个脑子里突然灵光一现想要做快速验证的人，都可以在小程序上开发出一个原型产品。
