---
title: OISP 本地服務器設定 API
tags: OISP
description: OISP Local Server Setup API
---

## 雲端會員取得授權碼

1. 瀏覽【[登入雲端會員](https://cc.smart-ehome.com/member/login)】
1. 還沒有帳號者請先【[註冊為會員](https://cc.smart-ehome.com/member/register)】
1. 一個會員可建立多部本地服務器，不需為每一服務器註冊獨立的會員帳號。
1. App 以 WebView 登入後在會員頁面，可用 WebView API 抓取到授權碼 (以下 [雲端服務系統註冊](#1-雲端服務系統註冊) 會用到): \
  `document.querySelector('#authcode').textContent`
![範例1](https://i.imgur.com/xbtX66Z.png)
1. 或使用【[取得會員授權碼 API](https://cc.smart-ehome.com/member/authcode)】, 返回為 JSON 物件:
![範例2](https://i.imgur.com/AMWiKrF.png)
1. 每次產生的授權碼有效時間為兩個小時，請在此期間內使用。逾時請刷新會員頁面，重新抓取授權碼使用。


## 搜尋本地服務器及初始化

1. 在區域網路中以 UDP【[尋找本地服務器](./Web%20API.md#app-尋找本地服務器)】。
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
3. 若 `server` 或 `s_id` 為空的則表示本地服務器尚未向雲端主機註冊，請繼續執行 [雲端服務系統註冊](#1-雲端服務系統註冊)。


## 管理本地服務器組態

+ [取得廠商通訊模組清單](#2-取得廠商通訊模組清單)
+ [取得現有模組組態](#3-取得現有模組組態)
+ [儲存現有模組組態](#4-儲存現有模組組態)
+ [測試通訊埠及偵測設備](#5-測試通訊埠及偵測設備)
+ [暫停執行模組](#6-暫停執行模組)
+ [啟動執行模組](#7-啟動執行模組)
+ [查詢執行模組](#8-查詢執行模組)


## Web API 一律採用 POST 方法

1. 為了安全起見 Web API 一律採用 POST 方法。
1. `POST` 資料內容可選擇下列兩種表示法，請於 HTTP 標題指定 `Content-Type`:\
    以 [雲端服務系統註冊](#1-雲端服務系統註冊) API 說明:
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
    ```json
    // 正確: status = 0 表示成功
    {
        "server": "<本地服務器UUID>",
        "status": 0,
        "payload": _視各功能返回不同型態的_物件_陣列_或_字串_
    }

    // 錯誤: status 非零: 見 <payload> 錯誤訊息
    {
        "server": "<本地服務器UUID>",
        "status": 1,
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
    + `<授權碼>`: 向系統管理者索取【[本地服務器的通行令牌](./Web%20API.md#系統管理者取得新用戶的授權碼-get_authcode)】。若為新的系統尚無系統管理者則可省略本欄位。
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

### 1. 雲端服務系統註冊

1. URI = [`<local.web_api>`](#json)`/setup/register`
    ```
    token=<通行令牌>
    CMauthcode=<雲端會員取得授權碼>
    name=<本地服務器名稱>
    region=<地區-可省略>
    server=<本地服務器UUID-可省略>
    s_id=<本地服務器短ID-可省略>
    ```
    + `token`: 通行令牌為溝通本地服務器的授權依據。
    + `CMauthcode`: 由 [雲端會員取得授權碼](#雲端會員取得授權碼) 第 4 或 5 項取得的 `authcode`。
    + `region`: 地區，目前以網頁瀏覽語系識別代碼 [Web browser language identification codes](https://www.metamodpro.com/browser-language-codes) 表示之。主要用於本地主機行事曆區別及 MQTT 系統發出訊息用。若未指定則以瀏覽器語系為主，參見 [Web API 訊息語系](./Web%20API.md#訊息語系-lang)。
    + `server` 欄位為 UUID 格式，若格式不符則由雲端系統給定。此欄位會原封不動在返回結果中 (和 `status` 同一層)，可用以識別是由哪個送出的請求。沒有特殊需求時可省略本欄由系統決定。
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
              "AMIoT-2G": {
                "drive": "AMIoT",
                "description": "8 迴路 DI + 8 迴路燈控模組",
                "connection": "tcp",
                "shared": false,
                "types": { "DO": 8, "DI": 8 },
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
                "connection": "tcp, serial",
                "shared": true,
                "types": { "DI": 8, "DO": 6, "PD": "DO" }
              },
              "AM-206": {
                "drive": "amma",
                "descriptions": "6 迴路 DO 模組",
                "connection": "tcp, serial",
                "shared": true,
                "types": { "DO": 6 }
              },
              "AM-210": {
                "drive": "amma",
                "descriptions": "10 迴路 DO 模組",
                "connection": "tcp, serial",
                "shared": true,
                "types": { "DO": 10 }
              },
              "AM-210S": {
                "drive": "amma",
                "descriptions": "10 迴路燈控模組",
                "connection": "tcp, serial",
                "shared": true,
                "types": { "DO": 10, "PD": "DO" }
              },
              "AM-250": {
                "drive": "amma",
                "descriptions": "2 迴路調光模組",
                "connection": "tcp, serial",
                "shared": true,
                "types": { "DM": 2 }
              },
              "AM-255": {
                "drive": "amma",
                "descriptions": "4 迴路調光模組",
                "connection": "tcp, serial",
                "shared": true,
                "types": { "DM": 4 }
              },
              "AM-260": {
                "drive": "amma",
                "descriptions": "6 迴路調光模組",
                "connection": "tcp, serial",
                "shared": true,
                "types": { "DM": 6 }
              },
              "AM-310": {
                "drive": "amma",
                "descriptions": "12 迴路燈控模組",
                "connection": "tcp, serial",
                "shared": true,
                "types": { "DO": 12, "PD": "DO" }
              },
              "AM-311": {
                "drive": "amma",
                "descriptions": "12 迴路 DI 模組",
                "connection": "tcp, serial",
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
                "connection": "tcp, serial",
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
        + `minor`: 增加、修正或調整功能，不影響原有的組態設定及運作。
        + `revision`: 錯誤的修正，不影響原有的組態設定及運作。
    + 其他欄位供參考用。

### 3. 取得現有模組組態

1. URI = [`<local.web_api>`](#json)`/setup/modules/fetch`
    ```
    token=<通行令牌>
    server=<本地服務器UUID>
    modules=<模組名稱清單-可省略>
    ```
    + `modules` 欄位可省略，表示取得現有系統的所有模組組態。
    + `<模組名稱清單>` 內容為一字串陣列表示要取得哪些目錄下的組態，如: "`amma,modbus,onvif`"，若無該目錄則不會有對應的返回值。通常不指定模組名稱清單，會抓取現有全部模組的組態。
    + 本系統會自動偵測本地區域網路上的 onvif 攝影機，此一過程約要花費 3 秒，請稍加等待。

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
                    "connector": {
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
        + 其餘 `json-file2, ...` 則視各驅動模組而有不同。通常以 `devices` (`.json`) 儲存此驅動模組所擁有的連線資訊。每一連線下有哪些實體設備、如何對應 (重組) 到 OISP 頁面 (設備/功能) 關係，則會另儲於 "`XX-config`.json" 下。
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
                "http://192.168.1.233:80/onvif/device_service": {
                    "name": "Embedded Net DVR",
                    "xaddr": "http://192.168.1.233:80/onvif/device_service"
                },
                "http://192.168.1.232:10080/onvif/device_service": {
                    "name": "Vstarcam",
                    "xaddr": "http://192.168.1.232:10080/onvif/device_service"
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

1. URI = [`<local.web_api>`](#json)`/setup/modules/testDevices` \
    請用 JSON 格式傳輸: `Content-Type: application/json`
    ```json
    {
        "token": "<通行令牌>",
        "server": "<本地服務器UUID>",
        "payload": [
            [ _config, Conn_Object, XX_Config ],
            //...
        ]
    }
    ```
    + `_config` 為之前的驅動模組物件，用到欄位: `{ vendor, drive }`。
    + `Conn_Object` 為連線必要資訊，視各驅動模組需求而有不同。
    + `XX_Config` 為各模組該連線所需的額外組態: 一般對應到該模組的 `XX-config.json` 內容; 但不是每個驅動模組都需要，在驅動器中指定了 `multiplex` 為 `false` 時就不用此欄位。一般而言，除非驅動模組有特別需求才需此欄位，否則可不用帶此欄。例如: 銨茂驅動模組預設偵測設備編號 1\~10，如要能偵測到編號 20，則要帶入 `{ "_numDevices": 20 }`，偵測數量越大則等待的回應時間會越長，每次的查詢約要耗時 0.2 秒，預設查詢到編號 10，約需時 2 秒。
    + 請注意: 系統攝影機模組`onvif` 的密碼欄位關鍵字為 `pass`，而非 `password`。
    + `綠米空調伴侣` 會自動偵測網關及子設備，偵測等待時間為 1 秒，只要指定 `_config` 即可。
    + 範例:
    ```json
    {
      "server": "<本地服務器UUID>",
      "token": "<通行令牌>",
      "payload": [
        [ { "vendor": "AMMA", "drive": "minix" },
          { "host": "192.168.1.114",
            "port": 16779,
            "user": "admin",
            "password": "password"
          }
        ],
        [ { "vendor": "AMMA", "drive": "amma" },
          { "type": "tcp",
            "host": "192.168.1.38",
            "port": 5300
          },
          { "_numDevices": 20 }
        ],
        [ { "vendor": "AMMA", "drive": "AMIoT" },
          {
            "type": "tcp",
            "host": "192.168.1.8",
            "port": 5088,
            "user": "AMMA",
            "password": "123456"
          }
        ],
        [ { "vendor": "OTHERS", "drive": "onvif" },
          [ { "name": "Embedded Net DVR",
              "xaddr": "http://192.168.1.233:80/onvif/device_service",
              "user": "admin",
              "pass": "password"
            },
            { "name": "Vstarcam",
              "xaddr": "http://192.168.1.232:10080/onvif/device_service",
              "user": "admin",
              "pass": "password"
            }
          ]
        ],
        [ { "vendor": "LUMI", "drive": "acpartner" }
        ]
      ]
    }
    ```

2. 返回:
    ```json
    {
      "server": "<本地服務器UUID>",
      "status": 0,
      "payload": [
        [ [2, 4, "AM-DIO"], [1, 8, "AM-210S"], [0, 9, "AM-CURTAIN"] ],
        [ [1, 7, "AM-250"], [2, 11, "AM-260"], [15, 1, "AM-100"] ],
        [ ["AMIoT-2G", "0021702B0042"] ]
        [ { "name": "Embedded Net DVR",
            "xaddr": "http://192.168.1.233:80/onvif/device_service",
            "user": "admin",
            "pass": "password",
            "type": "DS-7204HQHI-K1",
            "manufacturer": "Hangzhou Hikvision Digital Technology Co., Ltd",
            "serial_no": "0420171110CCWR128188354WCVU",
            "profiles": [
              { "name": "ProfileToken001",
                "rtsp": "rtsp://192.168.1.233:21080/Streaming/Unicast/channels/101",
                "resolution": [ 960, 480 ],
                "ptz": ""
              },
              { "name": "ProfileToken002",
                "rtsp": "rtsp://192.168.1.233:21080/Streaming/Unicast/channels/201",
                "resolution": [ 960, 480 ],
                "ptz": ""
              },
              { "name": "ProfileToken003",
                "rtsp": "rtsp://192.168.1.233:21080/Streaming/Unicast/channels/301",
                "resolution": [ 1920, 1080 ],
                "ptz": ""
              },
              { "name": "ProfileToken004",
                "rtsp": "rtsp://192.168.1.233:21080/Streaming/Unicast/channels/401",
                "resolution": [ 1920, 1080 ],
                "ptz": ""
              }
            ]
          },
          { "name": "Vstarcam",
            "xaddr": "http://192.168.1.232:10080/onvif/device_service",
            "user": "admin",
            "pass": "password",
            "type": "Hi3518eV100",
            "manufacturer": "Vstarcam",
            "serial_no": "VSTA481137EHDPL",
            "profiles": [
              { "name": "PROFILE_000",
                "rtsp": "rtsp://192.168.1.232:10554/tcp/av0_0",
                "resolution": [ 1280, 720 ],
                "ptz": "xyz"
              },
              { "name": "PROFILE_001",
                "rtsp": "rtsp://192.168.1.232:10554/tcp/av0_1",
                "resolution": [ 640, 360 ],
                "ptz": "xyz"
              }
            ]
          }
        ],
        [ { "gateways": [
              { "protocal": "UDP",
                "port": "9898",
                "sid": "50ec5XXXXXXX",
                "model": "acpartner.v3",
                "proto_version": "2.0.2",
                "ip": "192.168.1.223",
                "token": "dLKqn54LjXXXXXXX"
              }
            ],
            "xxconfig": [
              { "devList": {
                  "01": "50ec5XXXXXXX",
                  "02": "158d000XXXXXXX"
                },
                "devices": {
                  "01": {
                    "name": "空調伴侶",
                    "type": "acpartner.v3",
                    "icon_id": "087",
                    "functions": {
                      "01": {
                        "name": "電源",
                        "type": 3,
                        "value": [ { "=": [ { "0": "關" }, { "1": "開" } ] } ]
                      },
                      "02": {
                        "name": "模式",
                        "type": 3,
                        "value": [ { "=": [ { "0": "Heat ▼" }, { "1": "Cool ▼" }, { "2": "Auto ▼" }, { "3": "Dry ▼" }, { "4": "Wind ▼" }, { "5": "Circle ▼" } ] } ]
                      },
                      "03": {
                        "name": "風速",
                        "type": 3,
                        "value": [ { "=": [ { "0": "Low ▼" }, { "1": "Middle ▼" }, { "2": "High ▼" }, { "3": "Auto ▼" }, { "4": "Circle ▼" } ] } ]
                      },
                      "04": {
                        "name": "風向",
                        "type": 3,
                        "value": [ { "=": [ { "0": "Unswing" }, { "1": "Swing" } ] } ]
                      },
                      "05": {
                        "name": "設定溫度",
                        "type": 3,
                        "value": [ { "=": [ { "17": "17℃ ▼" }, { "18": "18℃ ▼" }, { "19": "19℃ ▼" }, { "20": "20℃ ▼" }, { "21": "21℃ ▼" }, { "22": "22℃ ▼" }, { "23": "23℃ ▼" }, { "24": "24℃ ▼" }, { "25": "25℃ ▼" }, { "26": "26℃ ▼" }, { "27": "27℃ ▼" }, { "28": "28℃ ▼" }, { "29": "29℃ ▼" }, { "30": "30℃ ▼" } ] } ]
                      },
                      "06": {
                        "name": "供電",
                        "type": 3,
                        "value": "2"
                      },
                      "07": {
                        "name": "光照度",
                        "type": 1,
                        "value": "0~1200"
                      },
                      "08": {
                        "name": "音效",
                        "type": 3,
                        "value": [ { "=": [ { "0": "停止播放" }, { "10000": "警車聲 ▼" }, { "1": "消防車聲 ▼" }, { "2": "警報聲_1 ▼" }, { "3": "警報聲_2 ▼" }, { "4": "鬼聲 ▼" }, { "5": "槍聲 ▼" }, { "6": "機槍聲 ▼" }, { "7": "防空警報聲 ▼" }, { "8": "狗叫聲 ▼" }, { "10": "叮咚 ▼" }, { "11": "敲木魚聲 ▼" }, { "12": "搞笑聲_1 ▼" }, { "13": "電話鈴聲_2 ▼" }, { "20": "輕快鈴聲 ▼" }, { "21": "節奏鈴聲 ▼" }, { "22": "撥弦樂器 ▼" }, { "23": "輕音樂 ▼" }, { "24": "敲擊樂器 ▼" }, { "25": "音效_1 ▼" }, { "26": "音效_2 ▼" }, { "27": "音效_3 ▼" }, { "28": "音效_4 ▼" }, { "29": "音效_5 ▼" } ] } ]
                      }
                    }
                  },
                  "02": {
                    "name": "無線開關（升級版）",
                    "type": "sensor_switch.aq3",
                    "icon_id": "071",
                    "functions": {
                      "01": {
                        "name": "按鍵",
                        "type": 1,
                        "value": [ { "=": [ { "0": "偵測中" }, { "1": "單擊" }, { "2": "雙擊" }, { "3": "搖一搖" } ] } ]
                      },
                      "02": {
                        "name": "電池電力",
                        "type": 1,
                        "value": "***mv"
                      }
                    }
                  }
                }
              }
            ]
          }
        ]
      ]
    }
    ```
    + `payload` 為一陣列物件，對應請求 `payload` 的結果。每一結果為:
        + false: 連線失敗，可能 `ip:port` 或帳密錯誤。
        + true: 連線成功 (不保證連接設備回饋正常)，但不支援或無法偵測到有哪些設備。
        + 陣列或物件: 表示連線成功，且能自動偵測連接的設備結果。此一結果視各驅動模組而有不同。
    + 銨茂模組 (含 minix) 返回為一陣列，每一元素為 `[設備id, 模組id, 模組型號]`。`設備id`: `0~255`，`模組id` 說明如下:
        模組id | 型號及內容
        :----:|----
         1 | AM-100        DMX 32 迴路
         2 | AM-210        DO:10
         3 | AM-310        PD:10
         4 | AM-DIO,MT-80  DI:8   DO:6  PD:6
         5 | AM-311        DI:12
         7 | AM-250 Dimmer CH-2
         8 | AM-210S DO CH-10 PD-10 DO-10
         9 | AM-260 Dimmer CH-6 (或 Curtain)
        10 | AM-260 Dimmer CH-6 (或 Audio)
        11 | AM-260 Dimmer CH-6
        12 | AM-255 Dimmer CH-4
    + AMIoT 模組返回為一陣列，每一元素為 `[設備type, MAC]`。`設備type` = `"AMIoT-編號(兩碼)"`，說明如下:
        類型 | 編號 (十位數) | 編號 (個位數) CH 數
        :----:|:----:|----
        DO      | 0 | 範圍 (1-9 A-Z)
        DI      | 1 | 範圍 (1-9 A-Z)
        DIDO    | 2 | 範圍 (1-9 A-Z)
        Dimmer  | 3 | 範圍 (1-9 A-Z)
        Switch  | 4 | 範圍 (1-9 A-Z)
        Gateway | 5 | 固定 (0)
        Curtain | 6 | 範圍 (1-9 A-Z)
    + `onvif` 系統攝影機模組 (`$01`): 為一陣列對應請求的返回結果，每一攝影機可能會有多組串流，分別代表不同攝像鏡頭 (頻道)、解析度或幀率 (畫質)。依 `onvif` 規格以 `profiles` 表示有多少串流，每一串流內容如下: `{ name, rtsp, resolution, ptz }`
        + `name`: profile 名稱，為識別依據。
        + `rtsp`: 串流連線位置。
        + `resolution`: 畫面解析度 (參考用)，為一陣列以兩個元表示寬和高，例如: `[1280, 720]` 表示 1280x720 像素。
        + `ptz`: 是否支援鏡頭轉動及伸縮，以 `x` 表示橫向轉動、`y` 表示縱向轉動、`z` 表示深度伸縮 (焦距拉近拉遠)。由於各設備廠家實作問題，不代表有支援就一定能正常操作。
    + `綠米空調伴侣` 返回只有一個物件的陣列，物件內容如下:
        + `gateways`: 為網關物件陣列，包含網關序號、型號等基本資訊。
        + `xxconfig`: 對應每一網關的 `xx-config.json` 組態，包含每一子設備的序號及功能。
        + 目前支援以下設備:
            型號 | 名稱 | 參數
            ----|:----:|----
            `acpartner.v3` | 空調伴侶 | `on_off_cfg`, `mode_cfg`, `ws_cfg`, `swing_cfg`, `temp_cfg`
            `plug` | 智能插座 | `channel_0`, `load_power`, `energy_consumed`
            `ctrl_86plug` | 墙壁插座 | `channel_0`, `load_power`, `energy_consumed`
            `ctrl_86plug.aq1` | 墙壁插座 | `channel_0`, `load_power`, `energy_consumed`
            `ctrl_ln1` | 牆壁開關(零火線單鍵版) | `channel_0`
            `ctrl_ln1.aq1` | 牆壁開關(零火線單鍵版) | `channel_0`
            `lctrl_ln2` | 牆壁開關(零火線雙鍵版) | `channel_0`, `channel_1`
            `ctrl_neutral1` | 牆壁開關(單火線單鍵版) | `channel_0`
            `ctrl_neutral2` | 牆壁開關(單火線雙鍵版) | `channel_0`, `channel_1`
            `curtain` | 智能窗簾 | `curtain_level`
            `lumi.ctrl_dualchn` | 雙路控制器 | `channel_0`, `channel_1`
            `dimmer.rgbegl01` | RGB調光控制器 | `power_status`, `light_rgb`, `light_level`
            `ctrl_hvac.aq1` | 空調溫控器 | `on_off_cfg`, `mode_cfg`, `ws_cfg`, `temp_cfg`, `env_temp`, `on_off_status`
            `airrtc.tcpecn01` | 空調溫控器 | `on_off_cfg`, `mode_cfg`, `ws_cfg`, `temp_cfg`, `env_temp`, `on_off_status`
            `sensor_magnet.aq2` | 門窗傳感器 | `window_status`, `battery_voltage`
            `sensor_motion.aq2` | 人體傳感器 | `lux`, `illumination`, `motion_status`, `battery_voltage`
            `weather` | 溫濕度傳感器 | `temperature`, `humidity`, `pressure`, `battery_voltage`
            `sensor_wleak.aq1` | 水浸傳感器 | `wleak_status`, `battery_voltage`
            `sensor_switch` | 無線開關 | `button_0`, `battery_voltage`
            `sensor_switch.aq2` | 無線開關 | `button_0`, `battery_voltage`
            `sensor_switch.aq3` | 無線開關(升級版) | `button_0`, `battery_voltage`
            `sensor_86sw1.aq1` | 86無線開關單鍵 | `button_0`, `battery_voltage`
            `sensor_86sw2.aq1` | 86無線開關雙鍵 | `button_0`, `button_1`, `dual_channel`
            `sensor_cube.aqgl01` | 魔方傳感器 | `cube_status`, `rotate_degree`, `detect_time`, `battery_voltage`


### 6. 暫停執行模組

URI = [`<local.web_api>`](#json)`/setup/modules/stop`

```
token=<通行令牌>
server=<本地服務器UUID>
```

返回:
```json
{
    "server": "<本地服務器UUID>",
    "status": 0,
    "payload": "stop oisp-modules"
}
```

### 7. 啟動執行模組

URI = [`<local.web_api>`](#json)`/setup/modules/start`

```
token=<通行令牌>
server=<本地服務器UUID>
```

返回:
```json
{
    "server": "<本地服務器UUID>",
    "status": 0,
    "payload": "start oisp-modules"
}
```
+ 若模組已在執行中，再次啟動模組並不影響現有執行中的模組，這次的命令視同無作用。

### 8. 查詢執行模組

URI = [`<local.web_api>`](#json)`/setup/modules/status`

```
token=<通行令牌>
server=<本地服務器UUID>
```

返回:
```json
{
    "server": "<本地服務器UUID>",
    "status": 0,
    "payload": 2
}
```
+ `payload` 返回為有多少模組正在執行中，若為 `0` 表示模組已被暫停執行。
