## VMware ESXi 6.5

1. 下載安裝:
    * 要先註冊帳號才能登入查看: [ESXi 6.5](https://my.vmware.com/en/group/vmware/evalcenter?p=free-esxi6)。
    * 第一次尚未註冊 ESXi 6 時會出現下載按鈕，點下會再次要求填寫資料，同意後才會進真正可下載頁面，並會出現授權碼 `License Information: VMware vSphere Hypervisor 6 License`，將之複製記錄下來，之後要登記入安裝後的系統中。暫不用下載此頁面的檔案。
    * 再到此下載最新的版本: [VMware vSphere Hypervisor (ESXi ISO) image 2017-07-27 | 6.5.0 U1 | 332.63 MB | iso](https://my.vmware.com/en/group/vmware/evalcenter?p=vsphere-6&PubCID=4917229)。
    * 將之燒錄成光碟或可開機 USB。
    * 安裝步驟可參考網頁: [VMware ESXi 6.5.0 Step-By-Step Installation](http://maxlabvm.blogspot.tw/2016/06/vmware-esxi-60-step-by-step-installation.html)。
    <br>

1. 到此下載 [最新的修補](https://my.vmware.com/group/vmware/patch#search): (可之後再做)
    * `Select a Product` → `ESXi (Embedded and Installable)` 和 `6.5.0`，按下 `Search`。
    * 選最新的版本 `ESXi650-201801001 Product:ESXi (Embedded and Installable) 6.5.0` 下載。
    * 更新步驟可參考網頁: [How to Install latest ESXi 6.5 Update 1 Patches](http://maxlabvm.blogspot.tw/2017/07/how-to-install-latest-esxi-65-update-1.html)。
    <br>

1. VMware ESXi 參考資源:
    * [下載 ESXi 6.5 及取得授權碼](http://www.virten.net/2016/11/free-esxi-6-5-how-to-download-and-get-license-keys/)
    * [VMware ESXi 6.5.0-Host Client 操作使用說明](https://ithelp.ithome.com.tw/articles/10184459)
    * [How to disable/enable the CIM agent on the ESX/ESXi host](https://kb.vmware.com/s/article/1025757)
        ```sh
        # 當某些偵測感應不到時, 在 VM shell 執行命令如下:
        esxcli system wbem set --enable true
        chkconfig sfcbd-watchdog on
        chkconfig sfcbd on
        /etc/init.d/sfcbd-watchdog start
        ```
    * [Lab VM Testing](http://maxlabvm.blogspot.tw/)
    <br>

1. 虛擬機管理經由瀏覽器操作:
    * 可在瀏覽器中直接開啟虛擬機控制台。
    * 亦可經由 VMRC (VM Remote Console) 插件開啟虛擬機控制台，可更方便操作硬體的變更，及臨時更換光碟機到 vm ISO 或本地的光碟機。
    * 在虛擬機網路設定完成後，亦可使用遠端桌面 (Remote Desktop) 連線操作，可方便虛擬機和本機檔案傳輸、或剪貼簿交換。
    <br>

1. Windows 2003 虛擬機擴大系統硬碟 (C:) 操作:
    * 下載 [Dell Disk Expansion ExtPart.exe](http://www.dell.com/support/home/tw/en/twdhs1/Drivers/DriversDetails?driverId=R64398)。
    * 操作過程影片 [How to extend the system partition of a Windows Server 2003 VM](https://www.youtube.com/watch?v=KQw7tLqMm_M)。
    <br>
