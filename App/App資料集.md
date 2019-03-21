

# APP資料集

 ![N|Solid](https://md.smart-ehome.com/amma/manual/img/appdata/1.png)

### Document
使用者專屬資料夾位於『Document』之下。
使用者專屬資料夾名稱以登入UserID轉ＭＤ5命名。

使用者專屬資料夾（以下以專屬資料夾稱之）基本會有資料夾如下 ：
 - 專屬資料夾/temp  : 使用精靈暫存區,完成後temp資料夾更名為*homeid*
 - 專屬資料夾/*homeid* 
 - 專屬資料夾/*homeid*/wizard  : 頁面正式組態檔（待）
 - 專屬資料夾/*homeid*/packag  : 頁面組態套用UI產生的頁面檔（待）

 - 帳號資料？account.json (local)
（待）

 ### Document/專屬資料夾
 - 首頁資料 ( homelist.json )
 記錄所有項目清單。

 ```json
{
	"homes":{
				"homeid":{  
					"name":"", 		//顯示名稱
					"icon":"", 		//顯示圖示
					"sort":"",
					"token":"",	
					"uiid":"",		//使用的UI樣式
					"video":[       
						{
						 "name":"",
						 "url":"",
						 "user":"",
						 "pwd":""
						},
						{
						 "name":"",
						 "url":"",
						 "user":"",
						 "pwd":""
						}
					]
				},
				"homeid":{
					"name":"",
					"icon":"",
					"sort":"",
					"token":"",
					"uiid":""
				}
			}
	,
	"key":""						// OISP API ClientKey
}
```


### Document/專屬資料夾
 - 裝置連線資訊 Account.txt,存所有帳號資訊
 加密：Base64 >  AES  CTR 
 解密 :  AES  CTR > Base64

 ```json
 // 正確
{
	"xxx-xxx-xxx-xxx-xxx": {		// <伺服器ID>
		"xxx-xxx":{					// <設備名稱>
			"uid": "<登入帳號>",
			"pwd": "<登入密碼>",
			"clientid": "<client_id>",
			"topic": {
				"pub": "to/",
				"sub": ["from/#", "to/<uid>", "to/<cid>"]
			}
		}
	}
}

 ```

 - 伺服器連線資訊 ServerInfo.txt（from server）
加密：Base64 >  AES  CTR 
解密 :  AES  CTR > Base64

 ```json
{
	"server": "xxx-xxx-xxx-xxx-xxx",
	"name": "xxx",
	"local": {
	"web_api": "https://dev.xxx.com/api",
	"mqtt_host": "dev.xxx.com",
	"mqtt_port": 1883,
	"mqtt_ssl_port": 8883
	},
	"inet": {
	"web_api": "https://oisp.xxx.com/api",
	"mqtt_host": "oisp.xxx.com",
	"mqtt_port": 1883,
	"mqtt_ssl_port": 8883
	}
}
 ```
 - moduleData  ModuleList.json(from server）

 - 模組設備功能名稱 moduleconfig.json (local)
 
 - 情境  sceneList.json (local)

 ```json
{
	"scenes": {
		"**情境ID**1": {
			"name": "**情境名稱**",
			"icon_id":"",
			"active": true,
			"delay0": 0,
			"mode": 1,
			"actions": [{
				"id": "**模組ID+設備ID+功能ID**",
				"delay0": 0,
				"value": "**值**"
				},{
				"id": "**模組ID+設備ID+功能ID**",
				"delay0": 0,
				"value": "**值**"
				}
			]
		},
		"**情境ID**2": {
			"name": "**情境名稱**",
			"icon_id":"",
			"active": true,
			"delay0": 0,
			"mode": 1,
			"actions": [{
				"id": "**模組ID+設備ID+功能ID**",
				"delay0": 0,
				"value": "**值**"
				},{
				"id": "**模組ID+設備ID+功能ID**",
				"delay0": 0,
				"value": "**值**"
				}
			]
		}
	}
}
```

 - 頁面清單  pageList.json (local)

 ```json
{
    "pages":{
		"pageid":{
            "name":"",
            "ico_id":"",
            "action":"" 
        },
		"pageid":{
            "name":"",
            "ico_id":"",
            "action":""
        }
	}
}

```
- 頁面內容 [pageid].json

 ```json
{
    "nodes":{
		"nodeid":{
            "caption":"",
            "action": "**模組ID+設備ID+功能ID**"
        },
		"nodeid":{
            "caption":"",
            "action": "**模組ID+設備ID+功能ID**"
        }
	}      
}

```	  

# Wizard
與內購相關頁面組態檔，整體內購檔案結構區分如下：

- xxx資料夾 (xxx 內購ID)
- icon 檔： xxx(內購ID) \ image \ icon \ *.png (< iconID >.png)
- 一般 png 檔： xxx(內購ID) \ image \ *.png
- window layout 檔：xxx(內購ID) \ default \ window.json

- layout 檔：xxx(內購ID) \ default \ *.json

## 透過精靈轉化後APP頁面檔
> 一個檔案就是一頁(父視窗),父視窗含有 Ｎ個按鍵 + N個子視窗（window 節點下以ID為屬性）; 子視窗含有 Ｎ個按鍵。

     父視窗（windows,functions）  	> N個按鍵
             			> N個子視窗 > N個按鍵
     注意：
	 (1)所有子視窗不得彼此重疊,並皆位於父視窗 
	 (2)所有子視窗應建立於子視窗群內(windows) 
	 (3)按鍵群內(functions),不得含window

> 視窗：每個視窗有ID,該視窗下有功能鍵群,當按鍵高度超出視窗高度，表示為垂直卷軸方式，捲軸彩視窗倍數增加（垂直捲軸為高度倍數、水瓶捲軸為寬度倍數）

> 視窗切換，按鍵換頁 ： 儲存於Action 屬性（一般為其屬性值＝ “模組｜設備｜功能” < palyload > ）
### APP 特定用定義如下(大寫表示))：
- 換頁：“APP|GO|< pageID >"
- 本頁視窗切換 ： “APP|POP|<視窗ID>”

```	json
{
	"x":0,
	"y":0,
	"w":0,
	"h":0,
	"picture":"image\\< deviceID >\\bg.png",
	"functions":{
		"< funID >":{
			"x":0,
			"y":0,
			"w":0,
			"h":0,
			"upimage":"image\\< deviceID >\\< funID >_up.png",
			"dnimage":"image\\< deviceID >\\< funID >_dn.png",
			"action":"xxx|xxx|xxx"
		},
		"< funID >":{
			"x":0,
			"y":0,
			"w":0,
			"h":0,
			"upimage":"image\\< deviceID >\\< funID >_up.png",
			"dnimage":"image\\< deviceID >\\< funID >_dn.png",
			"action":"xxx|xxx|xxx"
		}
	}
}

```	

含 WINDOW 頁面
```	json
{
	"x":0,
	"y":0,
	"w":0,
	"h":0,
	"picture":"image\\< deviceID >\\bg.png",
	"windows":{
		"w000< windowID >":{
			"x":0,
			"y":0,
			"w":0,
			"h":0,
			"picture":"",
			"functions":{
				"< funID >":{
					"x":0,
					"y":0,
					"w":0,
					"h":0,
					"upimage":"image\\< deviceID >\\< funID >_up.png",
					"dnimage":"image\\< deviceID >\\< funID >_dn.png",
					"action":"xxx|xxx|xxx"
				},
				"< funID >":{
					"x":0,
					"y":0,
					"w":0,
					"h":0,
					"upimage":"image\\< deviceID >\\< funID >_up.png",
					"dnimage":"image\\< deviceID >\\< funID >_dn.png",
					"action":"xxx|xxx|xxx"
				}
			}
		},
		"w001< windowID >":{
			"x":0,
			"y":0,
			"w":0,
			"h":0,
			"picture":"",
			"functions":{
				"< funID >":{
					"x":0,
					"y":0,
					"w":0,
					"h":0,
					"upimage":"image\\< deviceID >\\< funID >_up.png",
					"dnimage":"image\\< deviceID >\\< funID >_dn.png",
					"action":"xxx|xxx|xxx"
				},
				"< funID >":{
					"x":0,
					"y":0,
					"w":0,
					"h":0,
					"upimage":"image\\< deviceID >\\< funID >_up.png",
					"dnimage":"image\\< deviceID >\\< funID >_dn.png",
					"action":"xxx|xxx|xxx"
				}
			}
		}		
	},
	"functions":{
		"< funID >":{
			"x":0,
			"y":0,
			"w":0,
			"h":0,
			"upimage":"image\\< deviceID >\\< funID >_up.png",
			"dnimage":"image\\< deviceID >\\< funID >_dn.png",
			"action":"xxx|xxx|xxx"
			},
		"< funID >":{
			"x":0,
			"y":0,
			"w":0,
			"h":0,
			"upimage":"image\\< deviceID >\\< funID >_up.png",
			"dnimage":"image\\< deviceID >\\< funID >_dn.png",
			"action":"xxx|xxx|xxx"
		}
	}
}
```