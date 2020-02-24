
<style>
code.red {
  color: #FF0000 !important;
}
</style>


[[TOC]]


# OISP 開放式智慧系統平台

![系統示意圖](https://cacoo.com/diagrams/XIbHkJ4p7f6aNVdg-D0CE9.png)

![本地主機/智慧中樞/智慧網關示意圖](https://cacoo.com/diagrams/XIbHkJ4p7f6aNVdg-5F8D7.png)

-----

## 一、系統介紹

OISP (Open Intelligent System Platform) 是一全新的開放式智慧系統平台。在這個平台上開放了智慧家居控制、感測 (含 IoT) 等一系列的規格文件 (MQTT 及 Web API)、系統程式及實際範例，並已實做出手機 App 在 iOS 及安卓上可用。硬體廠商或系統整合商只需要專注為已有的硬體設備撰寫通訊模組 (驅動程式)、行銷及工程施工上，在軟體層面上不必花費太多心思即可讓用戶上手，可立即享受智慧家居帶來的便利性。

-----

## 二、系統特性

1. 不同廠商的硬體模組，可在本系統中以情境(群控)、智慧控制、排程中互動關聯，不再受限廠牌的拘束無法和其他硬件溝通。
2. 排程提供多種日期/星期/時間選擇，並可配合行事曆選擇上班或假日時執行哪些控制、情境或智慧模式等。
3. 開放性平台: 提供平台通訊協定、API 及系統原始碼，供各軟硬廠商、系統整合商或進階用戶參考，可開發各自的通訊模組，很容易的加入 OISP 系統中。
4. 新加入的廠商通訊模組，隨即可以在用戶端的 App 中找到立刻啟用操作。
5. 對於系統提供的 App 不滿意，可以依本系統的 MQTT 通訊協定及 Web API 自行開發不同的平台的應用程式。

### 更智慧的關聯互動控制

1. 本系統的智慧控制不僅是 `IFTTT` (IF This Then That) 那麼簡單，還可配合排程時間啟用，可設定多組控制命令及情境 (本系統並未限制數量)，只要系統主機的記憶體足夠且 CPU 夠力就可以。
2. 本系統的進階智慧控制採用狀態機模式，可用每一設備功能 (感應器狀態) 配合計時器來達到更複雜的應用場景。
3. 相關設定請參考【[App 智慧情境設定範例](https://md.smart-ehome.com/oisp/App/%E6%99%BA%E6%85%A7%E6%83%85%E5%A2%83.md)】。

* 【範例】當温度感測持續 5 分鐘超過 30 度，則觸發澆水情境，一小時後再檢查。還可以配合排程在上午九點啟動，下午五點停止本模式。
* 【範例】關懷長輩 (獨居):
    1. 檢查家中感測是否有人或燈光是否開啟，若是則表示長輩在家，結束本模式，否則進入下一狀態。
    2. 檢查兩小時內狀態未變更 (燈沒開啟、感測無人) 則推播 "通知: 長輩尚未回家"，並進入下一狀態。
    3. 當狀態有變更表示有人回來，推播 "通知: 長輩回家"，結束本模式。
    4. 搭配排程在晚上七點啟動「關懷長輩智慧控制」，則可關注晚間七到九點鐘長輩是否在家。
* 【範例】保全模式:
    1. 檢查前後門窗是否關閉，若未關閉則發出警告，30 秒後仍未關閉則離開保全模式。若都已關閉則推播 "保全已啟動"。
    2. 在保全模式中隨時偵測各感測器是否異常，若有異常則推播警報到 App 所在手機上。
    3. 在 30 秒內有兩個以上感測器異常則觸發警鈴十秒。
    4. 結束保全模式時推播 "保全已關閉"。

-----

## 三、未來系統擴展

1. 智慧音箱整合
2. Aqara (綠米) 智能家居 (已整合部份)
3. Apple Homekit
4. Goggle Home
5. Amazon Alexa (?)


-----

## 四、安裝工具 App 「Tools」

透過此工具App,「任何人」可以輕易建置OISP Local Server，完成此部分，相當於系統就建置完成了，工具App具有以下功能：
1. 註冊此服務器＆修改名稱
2. 設置服務器與選取使用的模組及其相關連接設置。
3. 定義虛擬設備與相關設備功能名稱。

**影片示範**
* 註冊＆設置名稱

<video src="https://www.smart-ehome.com/oisp/video/reg.mov" width="360" height="700" controls="controls">
</video>

* 新增模組運用與設置影像(支援ONVIF 監控系統)

<video src="https://www.smart-ehome.com/oisp/video/add_mods.mov" width="360" height="700" controls="controls">
</video>

-----

## 五、用戶端 App 「SmartHOME」

「SmartHOME」為使用者端App，當控制系統安裝完成後，可以透過精靈方式，STEP BY STEP 快速完成使用者控制介面。

==<code class="red">* 首位用戶即為管理者，日後其他App用戶則需要管理者分享給予授權。</code>==

###  iOS & Android

#### 新增專案教程
> 當控制系統安裝完成後，於該區域網路下進行新增專案。
可以手動或透過精靈方式，STEP BY STEP 快速完成第一次的使用者控制介面。

<video src="https://www.smart-ehome.com/oisp/video/build_ui.mov" width="360" height="700" controls="controls">
</video>

#### 新增含影像控制頁面教程

> 用戶可以自動一控制頁面，該頁面可以包含系統下的任何功能。

* 以下展示如何新增一個包含影像的控制頁面。

<code class="red">* 添加兩個影像以上，可以向左或右滑動視窗來切換影像。</code>
<video src="https://www.smart-ehome.com/oisp/video/add_page_video.mov" width="360" height="720" controls="controls">
</video>

#### 替換UI款式教程

> 用戶可以透過內購UI機制選用喜好的UI。

* 以下展示如何替換其他UI設計頁面。

<code class="red">* 添加兩個影像以上，可以向左或右滑動視窗來切換影像。</code>
<video src="https://www.smart-ehome.com/oisp/video/change_ui.mov" width="360" height="720" controls="controls">
</video>

#### 情境運用教程

> 透過此設置功能，您可以自訂新增一個情境的功能按鍵，當該按鍵按下後，依據所 > 設置的項目進行控制，達到一個所預定的情境狀態。

* 以下展示如何新增一個情境控制。

<video src="https://www.smart-ehome.com/oisp/video/scene.mov" width="360" height="720" controls="controls">
</video>

#### 排程運用教程

> 透過此設置功能，您可以自訂一個排程，並能於控制頁面中啟用或關閉該排程功能。
當該排程啟用後，依據所設置的條件時間符合，進行設置項目控制。

* 以下展示如何新增一個排程控制。
例： 我們要設置一個下課模式，當下課時關燈，上課時開啟電燈。
下課時間如下：
09:50 ~ 10:00、10:50 ~ 11:00、11:50 ~ 12:00、
14:50 ~ 15:00、15:50 ~ 16:00、16:50 ~ 17:00、
17:50 ~ 18:00

<video src="https://www.smart-ehome.com/oisp/video/schedule.mov" width="360" height="720" controls="controls">
</video>

#### 帳號管理
帳號分為管理者與一般用戶，一般用戶無法新增或刪除系統功能，系統功能包含情境、排程、智慧控制運用與推播功能。

<code class="red">* 一般用戶無權限新增或刪除：情境、排程、智慧控制運用與推播功能。</code>

##### 授權使用者為管理者或一般用戶
> 授權使用者帳號為管理者或一般用戶。
  一般用戶無法設置新增或異動系統功能。

* 以下展示如何授權用戶為管理者或一般用戶。

<video src="https://www.smart-ehome.com/oisp/video/change_user.mov" width="360" height="720" controls="controls">
</video>

##### 刪除使用者或該使用者下的裝置授權
> 刪除後該使用者或所刪除的裝置，將無法再連線控制。

* 以下展示如何刪除用戶的裝置授權。

<video src="https://www.smart-ehome.com/oisp/video/del_user.mov" width="360" height="720" controls="controls">
</video>

-----

## 六、系統開發文件

### 1. App
+ [App 處理流程](https://md.smart-ehome.com/oisp/%E7%B3%BB%E7%B5%B1%E6%9E%B6%E6%A7%8B%E8%A8%AD%E8%A8%88/%E6%B5%81%E7%A8%8B/App%20%E8%99%95%E7%90%86%E6%B5%81%E7%A8%8B.md)
+ [Web API](https://md.smart-ehome.com/oisp/%E7%B3%BB%E7%B5%B1%E6%9E%B6%E6%A7%8B%E8%A8%AD%E8%A8%88/%E9%80%9A%E8%A8%8A%E5%8D%94%E5%AE%9A/Web%20API.md)
+ [MQTT 通訊協定](https://md.smart-ehome.com/oisp/%E7%B3%BB%E7%B5%B1%E6%9E%B6%E6%A7%8B%E8%A8%AD%E8%A8%88/%E9%80%9A%E8%A8%8A%E5%8D%94%E5%AE%9A/MQTT%20%E9%80%9A%E8%A8%8A%E5%8D%94%E5%AE%9A.md)
+ [智慧家庭–UI定義＆設計規範](https://md.smart-ehome.com/oisp/App/UI%E5%AE%9A%E7%BE%A9%EF%BC%86%E8%A8%AD%E8%A8%88%E8%A6%8F%E7%AF%84-%E6%99%BA%E6%85%A7%E5%AE%B6%E5%BA%AD.md)

### 2. 系統設定
+ [本地服務器設定 API](https://md.smart-ehome.com/oisp/%E7%B3%BB%E7%B5%B1%E6%9E%B6%E6%A7%8B%E8%A8%AD%E8%A8%88/%E9%80%9A%E8%A8%8A%E5%8D%94%E5%AE%9A/OISP%E6%9C%AC%E5%9C%B0%E6%9C%8D%E5%8B%99%E5%99%A8%E8%A8%AD%E5%AE%9AAPI.md)

### 3. 通訊模組開發
+ [通訊轉接模組功能](https://md.smart-ehome.com/oisp/%E7%B3%BB%E7%B5%B1%E6%9E%B6%E6%A7%8B%E8%A8%AD%E8%A8%88/%E6%A8%A1%E7%B5%84%E5%8A%9F%E8%83%BD/%E9%80%9A%E8%A8%8A%E8%BD%89%E6%8E%A5%E6%A8%A1%E7%B5%84%E5%8A%9F%E8%83%BD.md)

