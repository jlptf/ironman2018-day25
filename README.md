## Day 25 - 保護你的系統：套用 SSL

### 本日共賞

* 認證、授權與加密
* 加入 SSL

### 希望你知道

* [k8s 不自賞 - 基礎篇](https://ithelp.ithome.com.tw/articles/10192193)

<br/>

#### 認證、授權與加密

認證 (Authentication)、授權 (Authorization) 與加密 (Encryption)，對於 Cloud Security 來說一直是很重要的議題。簡單來說

* 認證：確認是否為合法使用者 (Authenticated User)
* 授權：確認是否有權限 (Permission) 能存取
* 加密：確保資料在傳輸中的安全性

> 參考文章 [Understanding Authentication, Authorization, and Encryption](https://www.bu.edu/tech/about/security-resources/bestpractice/auth/)

針對部署在 k8s 中的應用程式，我們可以套用 SSL (Secure Sockets Layer) 來加強應用程式的安全性，今天將分享如何加入 SSL

<br/>

#### 加入 SSL

開始前，這裡先說明加入 SSL 的大略步驟

1. 建立自我簽署憑證 (self-signed certificates)
2. 建立 Secret 物件
3. 為應用程式加入 SSL

**建立自我簽署憑證**

##### 憑證選擇

要套用 SSL 需要有一份公正第三方簽署的憑證 (certificates) 而網路上可以找到很多 SSL 供應商提供不同等級與方案的憑證。當然也有提供免費 SSL 憑證的供應商，例如 [Letsencrypt](https://letsencrypt.org/), [sslforfree](https://www.sslforfree.com/) 等等。

付費的憑證效期長但是就是要付錢，而免費的憑證通常只會提供較短期 (三個月) 的期限，缺點是需要常常更新。需要使用哪一種方案請自行斟酌。

這裡用的憑證，是透過 openssl 產生的自我簽署憑證。雖無第三方認證機構的效力，不過使用上與第三方憑證無異。

首先透過指令建立金鑰與憑證

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt
```

* `req`：使用 X.509 Certificate Signing Request（CSR） Management 產生憑證
* `-x509`：建立自我簽署的憑證
* `-nodes`：不使用密碼保護
* `-days 365`：設定憑證期限為 365 天
* `-newkey rsa:2048`：產生新的 RSA 金鑰
* `-keyout`：金鑰名稱與儲存位置
* `-out`：憑證名稱與儲存位置

接著會提示輸入憑證相關資訊。因為是測試用，只要填入 Email Address 即可，其他可以直接按 Enter 忽略。順利的話會產生 tls.key 與 tls.crt 這兩個檔案。
接下來我們要將 key 與 crt 部署到 k8s 中，k8s 就可以利用這兩個檔案來啟用加密

**建立 Secret 物件**

透過指令建立 Secret 物件

```bash
$ kubectl create secret generic tls-certificate --from-file=./tls.key --from-file=./tls.crt
secret "tls-certificate" created
```

`--from-file`：會需要使用金鑰與憑證

觀察一下名為 `tls-certificate` 的 Secret 物件

```bash
$ kubectl describe secret tls-certificate
Name:         tls-certificate
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
tls.key:  1704 bytes
tls.crt:  1021 bytes
```

我們可以看到 `tls-certificate` 包含兩個 Data 欄位即 tls.key 與 tls.crt 。

**為應用程式加入 SSL**

接著我們需要部署可以提供 SSL 服務的應用程式。我已經預先做好了 docker 映像檔，位置在 `jlptf/ssl-nginx`。照著之後的說明，如果一切順利無誤，則透過瀏覽器可存取以下畫面。

![](https://ithelp.ithome.com.tw/upload/images/20171226/2010706243YVCynt0s.png)

> 這邊顯示 “不安全”是正常的，因為我們是使用自我簽署憑證，就好像詐騙集團自己寫了一張“我們是合法機構”的告示貼在門口的意思是一樣的。

底下我先來解釋一下在映像檔中做了哪些事情。

首先修改 nginx 的 default.conf 設定檔，讓 nginx 能夠支援 SSL

```
# default.conf

server{
listen 80 default_server;
listen [::]:80 default_server;

listen 443 ssl default_server;
listen [::]:443 ssl default_server;

ssl_certificate /etc/nginx/ssl/tls.crt;
ssl_certificate_key /etc/nginx/ssl/tls.key;
}
```

其中 `ssl_certificate` 與 `ssl_certificate_key` 分別是憑證與金鑰的名稱與存放的位置。在部署到 k8s 後，我們會需要將上述的 tls-certificate (Secret 物件) ，放置到對應的位置上即 `/etc/nginx/ssl/`，因此名稱與路徑都十分重要。

接著攥寫部署需要用到的 .yaml 檔，這邊需要兩個檔案，分別是 `deploy.ssl.yaml` 與 `service.ssl.yaml` 內容如下

```
# deploy.ssl.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: ssl-nginx
spec:
  replicas: 1
  template:
    metadata:
      labels: 
        app: ssl-nginx
    spec:
      containers:
      - name: ssl-nginx
        image: jlptf/ssl-nginx   <=== 使用預先做好的映像檔
        ports:
        - containerPort: 443   <=== SSL port 為 443
        volumeMounts:
        - name: nginx-ssl    <=== 將 Volume 掛載到 /etc/nginx/ssl 目錄下
          mountPath: /etc/nginx/ssl
          readOnly: true
      volumes:
      - name: nginx-ssl    <=== 掛載一個 Volume 並綁定名為 tls-certificate 的 Secret 物件
        secret:
          secretName: tls-certificate
```

大部分的內容已在前面文章解釋說明過，請參考之前文章內容。需要特別注意的是

* `image: jlptf/ssl-nginx`：記得換成支援 SSL 的映像檔
* `containerPort`：SSL 預設 port 為 443
* `volumes.secret.secretName`：將名稱為 tls-certificate 的 Secret 物件掛載成名稱為 nginx-ssl 的 volume
* `spec.containers.volumeMounts`：將名稱為 nginx-ssl 的 volume 掛載到 /etc/nginx/ssl 並設定成唯讀

```
# service.ssl.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: ssl-web
spec:
  type: NodePort
  selector:
    app: ssl-nginx
  ports:
    - protocol: TCP
      port: 443
      targetPort: 443
```
這裡的設定只要注意 selector, port 與 targetPort 即可

接下來執行指令部署物件

```bash
$ kubectl apply -f deploy.ssl.yaml
deployment "ssl-nginx" created

$ kubectl apply -f service.ssl.yaml 
service "ssl-web" created
```

查看新增的 Service

```bash
$ kubectl get service
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP         5d
ssl-web      NodePort    10.0.0.227   <none>        443:32525/TCP   1m
```

`ssl-web` 即為新增加的 Service 物件，我們可以透過 32525 port 存取 SSL server。還記得怎麼取得 minikube IP 嗎？取得 IP 後，打開瀏覽器輸入 https://[minikubeIP]:32525 就能存取有提供 SSL 服務的 nginx。

> 注意 port 會不同

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day25/](https://jlptf.github.io/ironman2018-day25/)