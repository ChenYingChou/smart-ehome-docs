---
title: OISP 本地服務器設定 API
tags: OISP
description: OISP Local Server Setup API
---

## 雲端會員取得授權碼

1. 瀏覽【[登入雲端會員](https://cc.smart-ehome.com/member/login)】
1. 還沒有帳號者請先【[註冊為會員](https://cc.smart-ehome.com/member/register)】
1. 一個會員可建立多部本地服務器，不需為每一服務器註冊獨立的會員帳號。
1. App 以 WebView 登入後在會員頁面，可用 WebView API 抓取到授權碼 (以下 [本地服務器初始化](#1-本地服務器初始化) 會用到): \
  `document.querySelector('#authcode').textContent`
![範例1](https://i.imgur.com/xbtX66Z.png)
1. 或使用【[取得會員授權碼 API](https://cc.smart-ehome.com/member/authcode)】, 返回為 JSON 物件:
![範例2](https://i.imgur.com/AMWiKrF.png)
1. 每次產生的授權碼有效時間為兩個小時，請在此期間內使用。逾時請刷新會員頁面，重新抓取授權碼使用。


## 搜尋本地服務器及初始化

1. 在區域網路中以 UDP【[尋找本地服務器](https://md.smart-ehome.com/oisp/系統架構設計/通訊協定/Web%20API.md#app-尋找本地服務器)】。
2. 解碼後得到<b id="json"></b>
    ```json
    {
        "server": "12345678-1234-1234-1234-123456789abc",
        "name": "AMMA-2F",
        "s_id": "AM2F",
        "local": {
            "web_api": "http://192.168.1.37/api",
            "mqtt_host": "192.168.1.37",
            "mqtt_port": 1883,
            "mqtt_ssl_port": 8883
        },
        "inet": {
            "web_api": "https://cc.smart-ehome.com/api",
            "mqtt_host": "cc.smart-ehome.com",
            "mqtt_port": 1883,
            "mqtt_ssl_port": 8883
        }
    }
    ```
3. 若 `server` 或 `s_id` 為空的則表示本地服務器尚未向雲端主機註冊，請繼續執行 [本地服務器初始化](#1-本地服務器初始化)。


## 管理本地服務器組態結構

+ [取得廠商通訊模組清單](#2-取得廠商通訊模組清單)
+ [取得現有模組組態](#3-取得現有模組組態)
+ [儲存現有模組組態](#4-儲存現有模組組態)
+ [測試通訊埠及偵測設備](#5-測試通訊埠及偵測設備)


## Web API 一律採用 POST 方法

1. 為了安全起見 Web API 一律採用 POST 方法。
1. `POST` 資料內容可選擇下列兩種表示法，請於 HTTP 標題指定 `Content-Type`:\
    以 [本地服務器初始化](#1-本地服務器初始化) API 說明:
    ```
    token=<通行令牌>
    authcode=<雲端會員授權碼>
    name=<本地服務器名稱>
    server=<本地服務器UUID>
    s_id=<本地服務器短ID-可省略>
    ```
    範例內容為:
    ```
    authcode=GHqZZIcH...XAUMmFmN
    name=My Smart Home
    server=?
    ```
    1. JSON: `Content-Type: application/json`
        ```json
        {
            "authcode": "GHqZZIcH...XAUMmFmN",
            "name": "My smart home",
            "server": "?"
        }
        ```
    1. Form Urlencoded: `Content-Type: application/x-www-form-urlencoded`
        ```
        authcode=GHqZZIcH...XAUMmFmN&name=My+smart+home&server=%3F
        ```
1. 服務器回覆一律為 JSON 格式 (`Content-Type: application/json; charset=utf-8`)\
    以下 JSON 格式中的註解僅為說明用，實際返回不包含這些註解:
    ```js
    // 正確
    {
        "server": "<本地服務器UUID>",
        "status": 0,                     	// 0:成功
        "payload": _視各功能返回不同型態的_物件_陣列_或_字串_
    }

    // 錯誤
    {
        "server": "<本地服務器UUID>",
        "status": 1,                        // 非零:錯誤, 見 <payload> 錯誤訊息
        "payload": "<錯誤訊息>"
    }
    ```
1. `token`: 通行令牌為溝通本地服務器的授權依據。若為新的系統尚未建立使用者 (App 手機未註冊本地服務器) 則可免本欄。否則需先取得 [本地服務器的通行令牌](#0-取得本地服務器的設定通行令牌)。


## API

### 0. 取得本地服務器的設定通行令牌

1. URI = [`<local.web_api>`](#json)`/setup/token`
    ```
    authcode=<授權碼>
    token=<舊通行令牌-選項>
    server=<本地服務器UUID>
    ```
    + `<授權碼>`: 向系統管理者索取【[本地服務器的通行令牌](https://md.smart-ehome.com/oisp/系統架構設計/通訊協定/Web%20API.md#系統管理者取得新用戶的授權碼-get_authcode)】。若為新的系統尚無系統管理者則可省略本欄位。
    + `token`: 本欄位為選項，若有時表示在舊通行令牌有效期間來換取新的通行令牌，此時可不理會 `authcode`。
    + `<本地服務器UUID>`: 請帶【[搜尋本地服務器](#搜尋本地服務器及初始化)】取得的 `server` 欄位值。
2. 成功時返回 `payload` 的 `token` 將應用於隨後 API 中:
    ```json
    {
        "server": "<本地服務器UUID>",
        "status": 0,
        "payload": {
            "token": "<通行令牌>",
            "expires_in": 28800
        }
    }
    ```
    + `expires_in`: 表示 `<通行令牌>` 的有效時間秒數。系統檢查通行令牌允許逾時五分鐘內視為有效。

### 1. 本地服務器初始化

1. URI = [`<local.web_api>`](#json)`/setup/init`
    ```
    token=<通行令牌>
    authcode=<雲端會員取得授權碼>
    name=<本地服務器名稱>
    server=<本地服務器UUID-可省略>
    s_id=<本地服務器短ID-可省略>
    ```
    + `token`: 通行令牌為溝通本地服務器的授權依據。
    + `server` 欄位為 UUID 格式，若格式不符則由雲端系統給定。此欄位會原封不動在返回結果中 (和 `status` 同一層)，可用以識別是由哪個送出的請求。沒有特殊需求時請給定 "`?`" 由系統決定。
    + `s_id` 欄位文數字 4 碼，若長度不符時由雲端系統給定。沒有特殊需求時可省略。主要用於 MQTT 通訊中，以節省傳輸量。
    + `s_id` 及 `server` 即使自行指定，但在雲端系統發現有衝突時會主動返回唯一的值。這些值會在返回結果的 `payload` 物件中。
2. 返回:
    ```json
    {
        "server": "?",
        "status": 0,
        "payload": {
            "s_id": "Ar2U",
            "server": "c75c7fe0-4cbd-40d4-ae3f-d3bec3e3a1a5",
            "name": "自訂名稱"
        }
    }
    ```
    + 若 `status` 不為零表示錯誤，`payload` 為錯誤訊息。
    + `status` 為零時，`payload` 中的 `s_id` 及 `server` 可能和送出請求的值不一樣，這些資訊會儲存在本地服務器中，以後再用 UDP 搜尋時就會有此值。

### 2. 取得廠商通訊模組清單

1. URI = [`<local.web_api>`](#json)`/setup/modules/list`
    ```
    token=<通行令牌>
    server=<本地服務器UUID>

    ```
2. 返回:
    ```json
    {
      "server": "<本地服務器UUID>",
      "status": 0,
      "payload": {
        "serials": [ "COM1", "COM2", "USB1", "USB2" ],
        "vendors": {
          "AMMA": {
            "descriptions": "AMMA eHome Smart Living",
            "homepage": "http://www.amma.com.tw",
            "drives": {
              "amma": {
                "descriptions": "銨茂模組",
                "date": "2019-10-01",
                "version": "1.0.0",
                "multiplex": true,
                "homepage": "",
                "author": "Sam Wang"
              },
              "minix": {
                "descriptions": "銨茂 MiniX 主機",
                "date": "2019-08-01",
                "version": "1.1.0",
                "multiplex": false,
                "homepage": "",
                "author": "Sam Wang, Tony Chen"
              },
              "AMIoT": {
                "descriptions": "銨茂 IoT",
                "date": "2019-10-01",
                "version": "1.0.0",
                "multiplex": true,
                "homepage": "",
                "author": "Sam Wang"
              }
            },
            "modules": {
              "AM-MiniX": {
                "drive": "minix",
                "descriptions": "AMMA MiniX 主機",
                "connection": "tcp",
                "shared": false
              },
              "AMIoT-0A": {
                "drive": "AMIoT",
                "descriptions": "10 迴路 DO 模組",
                "connection": "tcp",
                "shared": false,
                "types": { "DO": 10 }
              },
              "AMIoT-1K": {
                "drive": "AMIoT",
                "descriptions": "20 迴路 DI 模組",
                "connection": "tcp",
                "shared": false,
                "types": { "DI": 20 }
              },
              "AMIoT-36": {
                "drive": "AMIoT",
                "descriptions": "6 迴路調光模組",
                "connection": "tcp",
                "shared": false,
                "types": { "DM": 6 }
              },
              "AM-DIO": {
                "drive": "amma",
                "descriptions": "8 迴路 DI + 6 迴路燈控模組",
                "connection": "tcp, 485",
                "shared": true,
                "types": { "DI": 8, "DO": 6, "PD": "DO" }
              },
              "AM-206": {
                "drive": "amma",
                "descriptions": "6 迴路 DO 模組",
                "connection": "tcp, 485",
                "shared": true,
                "types": { "DO": 6 }
              },
              "AM-210": {
                "drive": "amma",
                "descriptions": "10 迴路 DO 模組",
                "connection": "tcp, 485",
                "shared": true,
                "types": { "DO": 10 }
              },
              "AM-210S": {
                "drive": "amma",
                "descriptions": "10 迴路燈控模組",
                "connection": "tcp, 485",
                "shared": true,
                "types": { "DO": 10, "PD": "DO" }
              },
              "AM-250": {
                "drive": "amma",
                "descriptions": "2 迴路調光模組",
                "connection": "tcp, 485",
                "shared": true,
                "types": { "DM": 2 }
              },
              "AM-255": {
                "drive": "amma",
                "descriptions": "4 迴路調光模組",
                "connection": "tcp, 485",
                "shared": true,
                "types": { "DM": 4 }
              },
              "AM-260": {
                "drive": "amma",
                "descriptions": "6 迴路調光模組",
                "connection": "tcp, 485",
                "shared": true,
                "types": { "DM": 6 }
              },
              "AM-310": {
                "drive": "amma",
                "descriptions": "12 迴路燈控模組",
                "connection": "tcp, 485",
                "shared": true,
                "types": { "DO": 12, "PD": "DO" }
              },
              "AM-311": {
                "drive": "amma",
                "descriptions": "12 迴路 DI 模組",
                "connection": "tcp, 485",
                "shared": true,
                "types": { "DI": 12 }
              }
            }
          },
          "LUMI": {
            "descriptions": "绿米联创科技",
            "homepage": "http://www.lumiunited.com/",
            "drives": {
              "aqara": {
                "descriptions": "AIoT 開放平台",
                "date": "2019-10-01",
                "version": "1.0.0",
                "multiplex": false,
                "homepage": "",
                "author": "Sam Wang"
              }
            },
            "modules": {
              "AQARA": {
                "drive": "aqara",
                "descriptions": "AIoT 開放平台",
                "connection": "web",
                "shared": false,
                "url": "https://aiot-oauth2.aqara.cn/authorize?client_id=1e2897xxxx13696&response_type=code&redirect_uri={redir}",
                "redir": "https://cc.smart-ehome.com/misc/aqara/{sid}"
              }
            }
          },
          "OTHERS": {
            "descriptions": "其他模組 (設定方式不同)",
            "homepage": "",
            "drives": {
              "onvif": {
                "descriptions": "ONVIF (開放式網路視訊介面論壇) 規格",
                "date": "2017-09-09",
                "version": "2.0.0",
                "multiplex": false,
                "homepage": "",
                "author": ""
              },
              "modbus": {
                "descriptions": "汎用型 Modbus 協定",
                "date": "2019-10-01",
                "version": "1.0.0",
                "multiplex": true,
                "homepage": "",
                "author": "Sam Wang"
              }
            },
            "modules": {
              "Onvif": {
                "drive": "onvif",
                "descriptions": "ONVIF (開放式網路視訊介面論壇) 規格",
                "connection": "tcp",
                "shared": false
              },
              "Modbus": {
                "drive": "modbus",
                "descriptions": "汎用型 Modbus 協定",
                "connection": "tcp, 485",
                "shared": true
              }
            }
          }
        }
      }
    }
    ```
    + `payload` 為各廠家模組清單，第一層為廠家名稱對應該廠家的所有模組。第二層為該廠家模組清單。
    + 第二層各模組清單中的 `version` 版號分為三組 "`major`.`minor`.`revision`":
        + `majon`: 有重大更新，可能和前一版有部份不相容。所接實體設備及組態會有差異，套用舊有設備及組態時可能會有問題。
        + `minor`: 增加、修正或調整功能，不影響原有的運作。
        + `revision`: 通常時指小錯誤的修正，不影響原有的運作。
    + 其他欄位供參考用。

### 3. 取得現有模組組態

1. URI = [`<local.web_api>`](#json)`/setup/modules/fetch`
    ```
    token=<通行令牌>
    server=<本地服務器UUID>
    modules=<模組名稱清單>
    ```
    + `modules` 欄位可省略，表示取得現有系統的所有模組組態。
    + `<模組名稱清單>` 內容為一字串陣列表示要取得哪些目錄下的組態，如: `["amma", "modbus", "onvif"]`，若無該目錄則不會有對應的返回值。通常不會特別指定模組名稱清單。

2. 返回:
    ```json
    {
        "server": "<本地服務器UUID>",
        "status": 0,
        "payload": {
            "目錄名稱": {
                "_config": {
                    "vendor": "AMMA",
                    "drive": "AMIoT",
                    "date": "2019-10-01",
                    "version": "1.0.0",
                    "activate": false
                },
                "devices": {
                    "version": 2,
                    "name": "AMIoT",
                    "devices": {
                        "1": { ... },
                        "2": { ... }
                    }
                },
                "01-config": { ... },
                "02-config": { ... }
            },
            //...
            "onvif": {
                "_config": {
                    "vendor": "OTHERS",
                    "drive": "onvif",
                    "date": "2017-09-09",
                    "version": "2.0.0",
                    "activate": true
                },
                "devices": {
                    "name": "攝影機",
                    "version": 0,
                    "devices": {}
                },
                "onvif": { ... }
            }
        }
    }
    ```
    + `payload` 欄位內容為現有各模組下的相關組態檔 (json 檔):
        + "`目錄名稱`": { json-file1, json-file2, ...}
        + 第一個 `josn-file1` 一定名為 "`_config`": 此為本模組所用的廠商驅動模組、相關日期/版號、及此模組是啟用或停用狀態。基本對應到【[2.取得廠商通訊模組清單](#2-取得廠商通訊模組清單)】的 `drives`，再加上 `activate` 欄位:
            ```json
            {
                "vendor": "廠家",
                "drive": "驅動模組名稱",
                "date": "2019-10-01",
                "version": "1.0.0",
                "activate": false
            }
            ```
        + 其餘 `json-file2, ...` 則視各驅動模組而有不同，通常有 `devices` ("devices.json") 儲存此驅動模組所擁有的連線資訊、有哪些實體設備、如何對應 (重組) 到 OISP 頁面 (設備/功能) 關係。
        + 每一 `json-file` 對應儲存在該模組目錄下的 "`json-file`.json"。
    + 特殊目錄:
        + "`sys`": 保留給系統模組 (`$00`)，為系統情境、智慧控制、排程、推播用。此系統模組不會在返回物件中。
        + "`onvif`": 為系統攝影機模組 (`$01`)。
    + `onvif` 模組組態如下:
        ```json
        {
            "_config": {
                "vendor": "OTHERS",
                "drive": "onvif",
                "date": "2017-09-09",
                "version": "2.0.0",
                "activate": true
            },
            "devices": {
                "name": "攝影機",
                "version": 0,
                "devices": {}
            },
            "onvif": {
                "http://192.168.1.233:88080/onvif/device_service": {
                    "name": "Embedded Net DVR",
                    "xaddr": "http://192.168.1.233:8080/onvif/device_service"
                },
                "http://192.168.1.232:19980/onvif/device_service": {
                    "name": "Vstarcam",
                    "xaddr": "http://192.168.1.232:19980/onvif/device_service"
                }
            }
        ```
        + `onvif` 欄位為系統自動偵測所得 (約花費 3 秒)，此例中偵測到兩組攝影機，每組攝影機串流數視各廠牌設備而不同。可用此資訊再呼叫【[5.測試通訊埠及偵測設備](#5-測試通訊埠及偵測設備)】API 取得詳細串流資訊，以供組態設定。

### 4. 儲存現有模組組態

1. URI = [`<local.web_api>`](#json)`/setup/modules/save`\
請用 JSON 格式傳輸: `Content-Type: application/json`
    ```json
    {
        "token": "<通行令牌>",
        "server": "<本地服務器UUID>",
        "payload": {
            "add": [ // ... 基本同 [3.取得現有模組組態] 的返回模組物件 (無目錄名稱)
                { 模組1 }, { 模組2 }, ...
            ],
            "delete": [
                "目錄名稱1", "目錄名稱2", ...
            ],
            "update": { // ... 同 [3.取得現有模組組態] 的返回模組物件
                "目錄名稱1": { 模組1 },
                "目錄名稱2": { 模組2 },
                // ...
            },
        }
    ```
    + `add` 新增模組: 由系統自行定義目錄名稱。每個模組必須包含廠家驅動模組 `_config` 內容。
    + `delete` 只須帶 "目錄名稱" 陣列即可，系統會停用該模組並刪除該目錄。
    + `update` 修改現有模組:
        + `_config` 組態中只有 `activate` 可變更，其他欄位請勿異動。
        + 通常以 `devices` 為主要組態來異動，並視需要增刪改其他相關的組態。

2. 返回:
    ```json
    {
        "server": "<本地服務器UUID>",
        "status": 0,
        "payload": "處理訊息"
    }
    ```
    + 請檢查 `status` 欄位，非零表示有錯 (可能部份有問題)。錯誤內容請參考 `payload`，其可能包含多個訊息，每一訊息為一行，以 CR/LF ("`\r\n`") 分隔。

### 5. 測試通訊埠及偵測設備

1. URI = [`<local.web_api>`](#json)`/setup/modules/testDevices`
    ```
    token=<通行令牌>
    server=<本地服務器UUID>
    payload=<模組名稱清單>
    ```
    + a
    + b

2. 返回:
    ```json
    {
        "server": "<本地服務器UUID>",
        "status": 0,
        "payload": {
            // !! todo
        }
    }
    ```
    + a
    + b

