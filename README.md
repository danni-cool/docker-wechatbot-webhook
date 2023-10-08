# docker-wechatbot-webhook

基于 [wechaty](https://github.com/wechaty/wechaty#readme) 和 [Express](https://github.com/expressjs/express) 开发

在微信和webhook机器人之间架一座桥梁，从此微信里也可以有自己的webhook机器人了，快用它集成到自己的自动化工作流中吧，推荐 [n8n](https://github.com/n8n-io/n8n)

![Docker Image Version (latest semver)](https://img.shields.io/docker/v/dannicool/docker-wechatbot-webhook) ![GitHub Workflow Status (with event)](https://img.shields.io/github/actions/workflow/status/danni-cool/docker-wechatbot-webhook/docker-build.yml)  ![Docker Pulls](https://img.shields.io/docker/pulls/dannicool/docker-wechatbot-webhook)

[view this project on docker hub :)](https://hub.docker.com/repository/docker/dannicool/docker-wechatbot-webhook/general)

## News (2023.10.8)
> 目前已知的是登录几天有几率会掉，应该是网页微信风控的问题（长时间无消息），目前解决方案是触发了掉线或者异常通知后，通知你配置的 `RECVD_MSG_API`，去处理扫码登录逻辑，比如访问暴露到外网的登录 api  http://localhost:3001/loginCheck?token=YOUR_PERSONAL_TOKEN。
如果有更好的方案可以和我交流 : )


## 一、启动

### 1. 本地调试

```
npm start
```

其他配置可以在 .env 文件中设置

```
# 如果想换端口
PORT=3001

# 如果想自己处理收到消息的逻辑，比如根据消息联动，在下面填上你的 API 地址, 默认为空
LOCAL_RECVD_MSG_API=https://example.com/your/url
```

### 2. Docker 部署

#### 拉取镜像

```bash
docker pull dannicool/docker-wechatbot-webhook
```

#### 启动容器

该方法会在后台启动一个 **只能给微信推消息** 的容器

```bash
docker run -d \
--name wxBotWebhook \
-p 3001:3001 \
dannicool/docker-wechatbot-webhook
```

####  启动容器可选参数

> Tips：需要增加参数使用 -e，多行用 \ 隔开，例如 -e  RECVD_MSG_API="https://example.com/your/url" \

| 功能 | 环境变量 | 案例 | 备注 |
|--|--|--|--|
|  收消息 API |   RECVD_MSG_API  |   RECVD_MSG_API="https://example.com/your/url"   |  如果想自己处理收到消息的逻辑，比如根据消息联动，填上你的处理逻辑 url，该行可以省略 |
| 自定义登录 API 令牌 | LOGIN_API_TOKEN | LOGIN_API_TOKEN=abcdefg123 | 容器启动后支持通过api 形式获得 登录状态 / 扫码登录 url，你也可以自定义一个自己的令牌，不配置的话，默认会生成一个 |

## 二、登录wx

以下只展示 docker 启动，本地调试可以直接在控制台找到链接

```bash
docker logs -f wxBotWebhook
```

找到二维码登录地址，图下 url 部分，浏览器访问，扫码登录wx

![](https://cdn.jsdelivr.net/gh/danni-cool/danni-cool@cdn/image/wechatlogindemo.png)

## 三、API

### 1. 推消息

- Url：<http://localhost:3001/webhook/msg>
- Methods: `POST`
- ContentType: `application/json`
- Body: 格式见下面表格

#### Body 参数说明

| 参数 |  说明 | 数据类型 | 默认值 | 可否为空 | 可选值 | 备注 |
|--|--|--|--|--|--|--|
| to | 会话名 | `String` |  |  N  |  | 发群消息填群名，发给个人填昵称 |
| isRoom | 是否发的群消息 | `Boolean` | `false`  | Y  |  `true` / `false`  | 这个参数决定了找人的时候找的是群还是人，因为昵称其实和群名相同在技术处理上  |
| type | 发送消息类型 | `String` | | N | `text` / `img` | 目前只支持 **文字** 和 **图片**，消息不支持图文自动拆分，请手动调多次  |
| content | 发送的消息 | `String` |  | N |  | 如果希望发多张图，type 指定为 img 同时，content 里填 url 以英文逗号分隔 |

#### Example（curl）
```bash
curl --location --request POST 'http://localhost:3001/webhook/msg' \
--header 'Content-Type: application/json' \
--data-raw '{
    "to": "testUser",
    "type": "text",
    "content": "Hello World!",
    "isRoom": false
}'
```

### 2. 收消息

> 收消息接口使用 `form` 传递参数，因为要兼容有文件的情况，文件目前也只兼容了**图片**

入参：

- Methods: `POST`
- ContentType: `multipart/form-data`
- Form格式如下

| formData |  说明 | 数据类型 | 可选值 |
|--|--|--|--|
| type | 表单类型 | `String` | `text` / `img` |
| content | 传输的内容,文件也放在这个字段，如果是图片收到的就是二进制buffer, 如果 `isSystemEvent` 为 '1', 将收到 `JSON String` | `String` / `Binary`  |  |
| source | 消息的相关发送方数据, JSON String | `String` | |
| isSystemEvent | 是否是来自系统消息事件（比如 上线 login，掉线 logout、异常事件 error）| `String` | 1 / 0

source 字段示例

```js
  {
    // 消息来自群，会有以下对象，否则为空字符串
    "room": {
      "id": "@@xxx",
      "topic": "abc" // 群名
      "payload": {
        "id": "@@xxxx",
        "adminIdList": [],
        "avatar": "xxxx", // 相对路径，应该要配合解密
        "memberIdList": [ //群里人的id
          "@xxxx",
          "@xxxx"
        ],
      },
      //以下暂不清楚什么用途，如有兴趣，请查阅 wechaty 官网文档
      "_events": {},
      "_eventsCount": 0,
    },


    // 消息来自个人，会有以下对象，否则为空字符串
    "to": {
        "id": "@xxx",

        "payload": {
            "alias": "", //备注名
            "avatar": "xxx",
            "friend": false,
            "gender": 1,
            "id": "@xxx",
            "name": "xxx",
            "phone": [],
            "signature": "hard mode",
            "star": false,
            "type": 1
        },

        "_events": {},
        "_eventsCount": 0,
      },

    // 消息发送方
    "from": {
      "id": "@xxx",

      "payload": {
        "alias": "",
        "avatar": "xxx",
        "city": "北京",
        "friend": true,
        "gender": 1,
        "id": "@xxxx",
        "name": "abc", //昵称
        "phone": [],
        "province": "北京",
        "star": false,
        "type": 1
      },

      "_events": {},
      "_eventsCount": 0,
    }

  }
```

### 3. 通过 API 获得登录状态

example: 访问登录shell 的 `http://localhost:3001/loginCheck?token=YOUR_PERSONAL_TOKEN`, 你将得到当前的登录态

token 是必填项，无需配置，初次启动项目会自动生成一个，当然你也可以配置一个简单好记的个人 token, 有两种方式

1. docker 启动，参数为 -e LOGIN_API_TOKEN="YOUR_PERSONAL_TOKEN"
2. `.env` 文件中，配置 LOCAL_LOGIN_API_TOKEN=YOUR_PERSONAL_TOKEN

> 如果都配置，docker 配置将覆盖本地配置

**请求体**

- Methods: `GET`
- URL: http://localhost:3001/loginCheck?token=YOUR_PERSONAL_TOKEN

**返回体**

| JSON |  说明 | 数据类型 | 可选值 |
|--|--|--|--|
| success | 登录成功与否 | `Boolean` | `true` / `false` |
| message | 当前登录用户名，登录失败将返回扫码登录URL  | `String`  |  |


## 四、更新日志

更新内容参见 [CHANGELOG](https://github.com/danni-cool/docker-wechat-roomBot/blob/main/CHANGELOG.md)
