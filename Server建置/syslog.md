## syslog

1. 在 Linux firewall 主機 /etc/rsyslog.d/ 下新增檔案 `debug.conf`:
    ```conf
    # Provides UDP syslog reception
    $ModLoad imudp
    $UDPServerRun 514

    user.debug				/var/log/debug
    ```

1. 重啟 rsyslog 服務:
    ```sh
    service rsyslog restart
    ```

1. 防火牆開啟允許誰連入使用 syslog udp port 514:
    ```sh
    # 或加入 clearos 的客製檔案中: /etc/clearos/firewall.d/custom
    iptables -t filter -I INPUT -s 允許的ip -p udp --dport 514 -j ACCEPT
    ```

1. 依 [RFC-5424](https://tools.ietf.org/html/rfc5424) 規定送出訊息: 以下以 [ABNF RFC-5234](https://tools.ietf.org/html/rfc5234) 表示
    ```bnf
    SYSLOG-MSG      = HEADER SP STRUCTURED-DATA [SP MSG]

    HEADER          = PRI VERSION SP TIMESTAMP SP HOSTNAME
                      SP APP-NAME SP PROCID SP MSGID
    PRI             = "<" PRIVAL ">"
    PRIVAL          = 1*3DIGIT ; range 0 .. 191
    VERSION         = NONZERO-DIGIT 0*2DIGIT
    HOSTNAME        = NILVALUE / 1*255PRINTUSASCII

    APP-NAME        = NILVALUE / 1*48PRINTUSASCII
    PROCID          = NILVALUE / 1*128PRINTUSASCII
    MSGID           = NILVALUE / 1*32PRINTUSASCII

    TIMESTAMP       = NILVALUE / FULL-DATE "T" FULL-TIME
    FULL-DATE       = DATE-FULLYEAR "-" DATE-MONTH "-" DATE-MDAY
    DATE-FULLYEAR   = 4DIGIT
    DATE-MONTH      = 2DIGIT  ; 01-12
    DATE-MDAY       = 2DIGIT  ; 01-28, 01-29, 01-30, 01-31 based on
                              ; month/year
    FULL-TIME       = PARTIAL-TIME TIME-OFFSET
    PARTIAL-TIME    = TIME-HOUR ":" TIME-MINUTE ":" TIME-SECOND
                      [TIME-SECFRAC]
    TIME-HOUR       = 2DIGIT  ; 00-23
    TIME-MINUTE     = 2DIGIT  ; 00-59
    TIME-SECOND     = 2DIGIT  ; 00-59
    TIME-SECFRAC    = "." 1*6DIGIT
    TIME-OFFSET     = "Z" / TIME-NUMOFFSET
    TIME-NUMOFFSET  = ("+" / "-") TIME-HOUR ":" TIME-MINUTE

    STRUCTURED-DATA = NILVALUE / 1*SD-ELEMENT
    SD-ELEMENT      = "[" SD-ID *(SP SD-PARAM) "]"
    SD-PARAM        = PARAM-NAME "=" %d34 PARAM-VALUE %d34
    SD-ID           = SD-NAME
    PARAM-NAME      = SD-NAME
    PARAM-VALUE     = UTF-8-STRING ; characters '"', '\' and
                                   ; ']' MUST be escaped.
    SD-NAME         = 1*32PRINTUSASCII
                      ; except '=', SP, ']', %d34 (")

    MSG             = MSG-ANY / MSG-UTF8
    MSG-ANY         = *OCTET ; not starting with BOM
    MSG-UTF8        = BOM UTF-8-STRING
    BOM             = %xEF.BB.BF
    UTF-8-STRING    = *OCTET ; UTF-8 string as specified
                      ; in RFC 3629

    OCTET           = %d00-255
    SP              = %d32
    PRINTUSASCII    = %d33-126
    NONZERO-DIGIT   = %d49-57
    DIGIT           = %d48 / NONZERO-DIGIT
    NILVALUE        = "-"
    ```

    * `VERSION` 目前 syslog 的版本為 1。

    * `PRIVAL` 的值是由 `<priority> = <facility>.<level>` 轉換而來:

        * `<facility>`

            | id | 值 || id | 值 |
            |:---:|:---:|-|:---:|:---:|
            | kern | 0 || ntp | 12 |
            | user | 1 || log audit | 13 |
            | mail | 2 || log alert | 14 |
            | daemon | 3 || clock daemon | 15 |
            | auth | 4 || local0 | 16 |
            | syslog | 5 || local1 | 17 |
            | lpr | 6 || local2 | 18 |
            | news | 7 || local3 | 19 |
            | uucp | 8 || local4 | 20 |
            | clock | 9 || local5 | 21 |
            | authpriv| 10 || local6 | 22 |
            | ftp | 11 || local7 | 23 |

        * `<level>`

            | id | 值 || id | 值 |
            |:---:|:---:|-|:---:|:---:|
            | emerg | 0 || warning | 4 |
            | alert | 1 || notice | 5 |
            | crit | 2 || info | 6 |
            | err | 3 || debug | 7 |

        * `PRIVAL = <facility> 值 x 8 + <level> 值`，例如: `user.debug` 的 `PRIVAL` = 1 x 8 + 7 = 15。

    * `TIMESTAMP` 以台北時區表示，例如: `2018-01-21T22:38:10+08:00`。

* 速成版組合完整訊息:
    ```
    PRI VERSION TIMESTAMP HOSTNAME APP-NAME PID MSGID - MSG

    PRI       = "<15>"                          ; user.debug = 1 x 8 + 7
    VERSION   = "1"                             ; 目前 syslog 支援版本為 "1"
    TIMESTAMP = "2018-01-21T22:38:10+08:00"     ; 取目前時間
    HOSTNAME  = "SAM-PC"                        ; 自己取名
    APP-NAME  = "my-prog"                       ; 自己取名
    PID       = "123"                           ; 省略則用 "-"
    MSGID     = "456"                           ; 省略則用 "-", syslogd 並未輸出此值

    組合送出訊息如下: (注意欄位間只能有一個空白，除了 PRI 和 VERSION 沒空白外)
    "<15>1 2018-01-21T22:38:10+08:00 SAM-PC my-prog 123 456 - message in here"

    註: MSG 中若包含有中文則必須用 UTF-8 編碼且要加 BOM，參見 MSG-UTF8 的定義。
    ```

* 客端操作:
    ```
    # 以 netcat 示範送出操作:
    prompt> nc -u 59.124.5.205 514
    <15>1 2018-01-21T22:38:10+08:00 hostname APP-NAME PID MSGID - message 1
    <15>1 2018-01-21T22:38:10+08:00 SAM-PC my-prog 123 456 - message 2
    <15>1 2018-01-21T22:38:10+08:00 SAM-PC my-prog - - - message 3
    ^C (中止)
    ```

* 伺服器端收到: 參考 /etc/rsyslog.d/debug.conf 中只接收 `user.debug`
    ```
    prompt> tail -f /var/log/debug
    Jan 21 22:38:10 hostname APP-NAME[PID] message 1
    Jan 21 22:38:10 SAM-PC my-prog[123] message 2
    Jan 21 22:38:10 SAM-PC my-prog message 3
    ```
