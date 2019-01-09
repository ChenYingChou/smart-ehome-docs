
# Push



```js
 URL  :  http://www.smart-ehome.com/push/SmartHOME/SmartHOME.php
```


# Keys For Notification
|Parameter|Usage|Description|
|--|--|--|
|token|與topic擇一|單一裝置推播token|
|topic|與token擇一|主題式推播|
|sound| |App resource 音效檔名<span style="color:red;background-color:yellow">未滿 Android 8.0 使用</span> （需要副檔名,因iOS需要）
|title|Optional|如果沒有，以日期時間自動取代
|body| |The notification's body text.


# Keys For Data
|Parameter|Usage|Description|
|--|--|--|
|ch||音效通道 <span style="color:red;background-color:yellow">(Android 8.0 以上使用) </span>
|space|Optional|Server ID|
|page|Optional|Page ID|


# ch 對應表
App 第一次安裝啟動時以系統語系註冊符合的語音檔於同通道ID。
＊若要異動，需移除App,重新安裝。

|ch ID|名稱|Description|
|--|--|--|
|0|新增關注|Follow|
|1|移除關注|Unfollow|
|2|入侵警報|Intrusion|
|3|火災警報|Fire|
|4|瓦斯洩漏|Gas|
|5|一氧化碳|CO|
|6|緊急呼叫|Emergency|
|7|警報|Alarm|
|8|保全啟動|Security ON|
|9|保全關閉|Security OFF|
|10|迴路異常|Loop , error|
|alarm|警鈴|Alarm|
|notice|通知|Notice|

#  sound 對應表


|中文|英文|名稱|Description|
|--|--|--|--|
|c0.mp3|e0.mp3|新增關注|Follow|
|c1.mp3|e1.mp3|移除關注|Unfollow|
|c2.mp3|e2.mp3|入侵警報|Intrusion|
|c3.mp3|e3.mp3|火災警報|Fire|
|c4.mp3|e4.mp3|瓦斯洩漏|Gas|
|c5.mp3|e5.mp3|一氧化碳|CO|
|c6.mp3|e6.mp3|緊急呼叫|Emergency|
|c7.mp3|e7.mp3|警報|Alarm|
|c8.mp3|e8.mp3|保全啟動|Security ON|
|c9.mp3|e9.mp3|保全關閉|Security OFF|
|c10.mp3|e10.mp3|迴路異常|Loop , error|
|alarm.mp3|alarm.mp3|警鈴|Alarm|
|notice.mp3|notice.mp3|通知|Notice|

# Sample

## 單一裝置

>  🧯 火災警報 🔥 (中文)

```js
http://www.smart-ehome.com/push/SmartHOME/SmartHOME.php?token=xxxxx&body=火災警報&ch=3&sound=c3.mp3
```


## 主題式推播

>  🧯 火災警報 🔥 (英文)

```js
http://www.smart-ehome.com/push/SmartHOME/SmartHOME.php?topic=xxxxx&body=Fire%20Alarm&ch=3&sound=e3.mp3
```