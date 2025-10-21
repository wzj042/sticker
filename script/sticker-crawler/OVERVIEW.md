
# 目标

1. 抓取微信表情包下载、发送量数据。
2. 基于 cloudflare worker 实现一个定时任务，每天定时抓取数据存储到 D1 数据库。
3. 抓取完数据发送 email 说明今日数据量变化趋势。

# 抓取逻辑

使用账号密码登陆，拼接返回的 cookies 用于后续请求。
先获取首页数据确定表情包列表，再根据表情包id获取历史数据，根据历史记录增量更新（表情包发送下载量和表情发送量）。

## 数据结构

# mediaUrl 拼接 Icon 地址
表情包(id, name, mediaUrl, dailyDownloadNum, dailySendNum, totalDownloadNum, totalSendNum)
# mediaUrl 拼接 Pic 地址
表情(id, name, mediaUrl, dailySendNum, stickerId)
表情包数据历史(id, date, downloadNum, sendNum, stickerId)
表情数据历史(id, date, sendNum, emojiId)


# 数据源结构

## 登录

> 获取登录 cookies, 除该接口外所有接口均需要传入登录 cookies。

api: https://sticker.weixin.qq.com/cgi-bin/mmemoticon-bin/login

header 参考：

```js
const headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept': 'application/json, text/plain, */*',
    'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
    'Origin': 'https://sticker.weixin.qq.com',
    'Referer': 'https://sticker.weixin.qq.com/'
}
```

form data 传递 email 和 pwd。

获取返回 header 的 set-cookie 字段，拼接为 cookies 用于后续请求。

## 首页数据

api: https://sticker.weixin.qq.com/cgi-bin/mmemoticon-bin/home?lang=zh_CN&f=json

###　主要数据结构


```json
{
  "base_resp": {
    // 返回码，0 表示成功
    "ret": 0,
    "err_msg": string,
  },
  "List": [
    // 表情包数据概览
    {
      "StikerID": string, // 表情包唯一标识
      "Name": string, // 表情包名称
      "DownloadNum": number, // 今日下载量
      "SendNum": number, // 今日发送量
      "DataTime" string, // 数据更新时间, e.g. Wed, 15 Oct 2025 16:00:00 GMT, 需要处理为 YYYY-MM-DD HH:MM
      "Icon": string, // 表情包图标后缀，需要拼接接口传入 Cookies 请求才能获取图片数据 https://sticker.weixin.qq.com/cgi-bin/mmemoticon-bin/getmedia?fileid=<Icon>
      "TotalDownloadNum":string, // 总下载量
      "TotalSendNum":string, // 总发送量
    },
    …
  ]
}
```

## 历史数据

> 获取表情粒度的今日发送量和表情包的30日历史数据

api: https://sticker.weixin.qq.com/cgi-bin/mmemoticon-bin/stikerreport?stikerid=<sticker_id>&f=json

### 主要数据结构

```json
{
  "base_resp": {
    // 返回码，0 表示成功
    "ret": 0,
    "err_msg": string,
  },
 "Report": [
    {
      "Pic": string, // 表情图片编码地址，需要拼接接口传入 Cookies 请求才能获取图片数据 https://sticker.weixin.qq.com/cgi-bin/mmemoticon-bin/getmedia?fileid=<Pic>
      "SendNum": number, // 今日发送量
      "DataTime": string, // 数据更新时间, e.g. 10月15日, 需要处理为 YYYY-MM-DD
      "Md5": string, // 表情唯一标识
    },
    …
  ],
  "History": [
    {
        "SendNum": number, // 发送量
        "DownloadNum": number, // 下载量
        "DateTime": string, // 数据更新时间, e.g. 20250916, 需要处理为 YYYY-MM-DD
    }, 
    …
  ]
}
```



# 术语

- 表情包（Sticker）：微信表情包，包含多个表情，使用`sticker_id`唯一标识。
- 表情（Emoji）：每个表情包可以包含多个表情，使用`md5`唯一标识。