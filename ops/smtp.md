# 搭建自己的 SMTP 郵件發送服務器

## 序言

SMTP 可以直接購買雲廠商的服務，比如 :

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [阿里雲郵件推送](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

也可以自己搭建郵件服務器 —— 發送不限量，綜合成本低。

下面，我們一步一步的演示如何自建郵件服務器。

## 服務器選購

自託管的 SMTP 服務器需要開放了 25、456、587 端口的公網 IP。

常用的公有云默認都封禁了這些端口，發工單可能可以開通，但終歸很麻煩。

我建議從那些開放了這些端口並支持設置反向域名的主機商中選購。

在這裡，我推薦下 [Contabo](https://contabo.com) 。

Contabo 是總部位於德國慕尼黑的主機提供商，成立於 2003 年，價格極具競爭力。

購買幣種選擇歐元，價格會更便宜 (8GB 內存 4 CPU 的服務器約 529 元人民幣每年，買一年免初裝費)。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

下單時備註 `prefer AMD`, 選用 AMD CPU 的服務器，性能更佳。

後文，我將以 Contabo 的 VPS 為例，演示如何搭建自己的郵件服務器。

## Ubuntu 系統配置

這裡操作系統選用 Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

如果 ssh 上服務器顯示 `Welcome to TinyCore 13!`（如下圖），表示這時候系統還沒裝完，請斷開 ssh，等待幾分鐘再次登錄。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

出現 `Welcome to Ubuntu 22.04.1 LTS` 就是初始化完成，可以繼續下面的步驟了。

### [可選] 初始化開發環境

這一步是可選的。

為了方便，我把 ubuntu 軟件的安裝、系統配置放到了 [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu)。

運行以下指令可一鍵安裝。

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

中國用戶請改用下面的指令，會自動設置語言、時區等。

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo 啟用 IPV6

啟用 IPV6 ，這樣 SMTP 也可以發送 IPV6 地址的郵件了。

編輯 `/etc/sysctl.conf`

修改或加入下面幾行

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

接下來參考 [contabo 教程 : 將 IPv6 連接添加到您的服務器](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

編輯 `/etc/netplan/01-netcfg.yaml`, 加上如下圖所示幾行 ( Contabo VPS 默認配置文件已經有這些行，取消註釋即可 )。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

然後 `netplan apply` 讓修改後的配置生效。

配置成功後，可使用 `curl 6.ipw.cn` 查看自己外網的 ipv6 地址。

## 克隆配置倉庫 ops

```
git clone https://github.com/wactax/ops.soft.git
```

## 生成域名 免費 SSL 證書

發送郵件需要 SSL 證書加密和簽名。

我們使用 [acme.sh](https://github.com/acmesh-official/acme.sh) 來生成證書。

acme.sh 是開源的自動化證書籤發工具，

進入配置倉庫 ops.soft ，運行 `./ssl.sh`，會在**上一級目錄**創建 `conf` 文件夾。

從 [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) 中找到你的 DNS 服務商，編輯 `conf/conf.sh` 。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

然後運行 `./ssl.sh 123.com`，就可以為你的域名生成 `123.com` 以及`*.123.com` 證書。

首次運行會自動安裝[acme.sh](https://github.com/acmesh-official/acme.sh)，並添加自動續期的定時任務。`crontab -l` 就可以看到，有如下這麼一行。

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

生成的證書的路徑類似 `/mnt/www/.acme.sh/123.com_ecc。`

證書續期會調用 `conf/reload/123.com.sh` 腳本，編輯這個腳本，可以添加諸如 `nginx -s reload` 的指令來刷新相關應用的證書緩存。

## 用 chasquid 搭建 SMTP 服務器

[chasquid](https://github.com/albertito/chasquid) 是 Go 語言編寫的開源 SMTP 服務器。

作為 Postfix、Sendmail 這些古老的郵件服務器程序的替代品，chasquid 更加簡單易上手，也更容易二次開發。

運行 chasquid 配置倉庫中的 `./chasquid/init.sh 123.com` 會全自動一鍵安裝（替換 123.com 為你的發信域名）。

## 配置郵件簽名 DKIM

DKIM 用於發送郵件的簽名，避免信件被當成垃圾郵件。

命令成功運行後，會提示你去設置 DKIM 記錄（如下圖）。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

到你的 DNS 添加 TXT 記錄即可（如下圖）。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## 查看服務狀態 & 日誌

 `systemctl status chasquid` 查看服務狀態。

正常運行時狀態如下圖

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` 或者 `journalctl -xeu chasquid` 可以查看出錯日誌。

## 反向域名配置

反向域名是讓 IP 地址可以解析為對應的域名。

設置反向域名，能避免郵件被識別為垃圾郵件。

當郵件接收後，接收服務器將對發送服務器的 IP 地址進行反向域名解析，以確認發送服務器是否具有有效的反向域名。

如果發送服務器沒有反向域名或者反向域名與發送服務器的 IP 地址不匹配，接收服務器可能會將該郵件識別為垃圾郵件或拒絕接收。

訪問 [https://my.contabo.com/rdns](https://my.contabo.com/rdns)，配置如下圖

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

設置完反向域名後，記得配置此域名 ipv4 和 ipv6 的正向解析到該服務器。

## 編輯 chasquid.conf 的 hostname

修改 `conf/chasquid/chasquid.conf` 為反向域名的值。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

接著運行 `systemctl restart chasquid` 重啟服務。

## 備份 conf 到 git 倉庫

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

比如我備份 conf 文件夾到自己的 github 流程如下

首先創建私有倉庫

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

進入 conf 目錄，然後提交到倉庫

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## 添加發件用戶

運行

```
chasquid-util user-add i@wac.tax
```

可以添加發件用戶

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### 效驗密碼是否設置正確

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

添加完成用戶 ,  `chasquid/domains/wac.tax/users` 會有更新，記得提交到倉庫。

## DNS 添加 SPF 記錄

SPF ( Sender Policy Framework ) 是一種郵件驗證技術，用於防範電子郵件欺詐。

它通過檢查發件人的 IP 地址與其所聲稱的域名的 DNS 記錄是否匹配來驗證郵件發送者的身份，從而防止欺詐者發送偽造的電子郵件。

添加 SPF 記錄，能儘可能避免郵件被識別為垃圾郵件。

如果你的域名服務器不支持 SPF 類型，添加 TXT 類型記錄即可。

比如，`wac.tax` 的 SPF 如下

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

`_spf.wac.tax` 的 SPF

`v=spf1 a:smtp.wac.tax ~all`

注意，我這裡有 `include:_spf.google.com`, 這是因為後續我將在谷歌郵箱配置用 `i@wac.tax` 做發信地址。

## DNS 配置 DMARC

DMARC 是（Domain-based Message Authentication, Reporting & Conformance）的縮寫。

它用於獲取 SPF 退信的情況（可能是配置錯誤導致，也可能是別人在冒充你發送垃圾郵件）。

添加 TXT 記錄 `_dmarc`，

內容如下

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

各個參數含義如下

### p (Policy)

表示如何處理未通過 SPF (Sender Policy Framework) 或 DKIM (DomainKeys Identified Mail) 驗證的郵件。p 參數可以設置為以下三個值之一：

* none：不採取任何行動，僅將驗證結果通過郵件報告機制反饋給發件人。
* quarantine：將未通過驗證的郵件放入垃圾郵件文件夾，但不會直接拒絕郵件。
* reject：直接拒絕未通過驗證的郵件。

### fo (Failure Options)

指定報告機制返回的信息量。它可以設置為以下值之一 :

* 0：報告所有郵件的驗證結果
* 1：僅報告未通過驗證的郵件
* d：僅報告域名驗證失敗
* s：僅報告 SPF 驗證失敗
* l：僅報告 DKIM 驗證失敗

### rua & ruf

* rua（Reporting URI for Aggregate reports）：接收聚合報告的郵件地址
* ruf（Reporting URI for Forensic reports）：接收詳細報告的郵件地址

## 添加 MX 記錄轉發來信到谷歌郵箱

因為沒找到免費的支持萬能地址 （Catch-All，能接收任何發送到此域名的郵件，不限制前綴）的企業郵箱，所以我用 chasquid 把所有的郵件都轉發到 我的 Gmail 信箱。

**如果你有自己付費的企業郵箱，請不要修改 MX，並跳過這一步。**

編輯 `conf/chasquid/domains/wac.tax/aliases`, 設置轉發郵箱

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` 表示所有郵件，`i` 是上面創建的發信用戶郵箱前綴。想轉發郵件，每個用戶都需要添加一行。

然後添加 MX 記錄即可 (我這裡直接指向反向域名的地址，如下圖第一行) 。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

配置完成後，可以用其他郵箱發件到 `i@wac.tax` 和 `any123@wac.tax` 看看是否能在 Gmail 收到信件。

如果不能收到，請檢查 chasquid 的日誌 (`grep chasquid /var/log/syslog`)。

## 用谷歌郵箱發送 i@wac.tax 的郵件

谷歌郵箱收件之後，自然希望回信也用 `i@wac.tax` 而不是 i.wac.tax@gmail.com 。

訪問 [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) ，點擊 『添加其他電子郵件地址』。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

接著，輸入被轉發到的郵箱收到的驗證碼即可。

最後，可以將其設置為默認發信地址（同時選擇以相同地址回覆）。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

如此，我們就完成了 SMTP 發信服務器的搭建以及同時用谷歌郵箱收發郵件。

## 發送測試郵件，檢查配置是否成功

進入 `ops/chasquid`

運行 `direnv allow` 安裝依賴 (前文一鍵初始化過程中已經安裝 direnv 並且在 shell 添加了 hook)

然後運行

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

參數含義如下

* user: SMTP 的用戶名
* pass: SMTP 的密碼
* to: 收件人

就可以發送測試郵件了。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

建議用 Gmail 收測試郵件，方便查看各項配置是否成功。

### TLS 標準加密

如下圖，有這個小鎖，表示 SSL 證書已經成功啟用。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

然後點擊『顯示原始郵件』

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

如下圖，Gmail 原始郵件頁面顯示 DKIM，即為 DKIM 配置成功。

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

檢查原始郵件頭部的 Received，還可以看到發信地址是 IPV6，這表示 IPV6 也配置成功了。
