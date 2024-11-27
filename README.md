# ntust-wg-easy
 用台科校內電腦架設個人用VPN服務 \
 本教學透過 zerotier + wireguard 實現連線
## 閱前聲明
學校近幾年因為資安問題，將校外直連校內ip用防火牆阻擋了 \
如果要存取校內網路，台科校方有提供VPN伺服器，請參閱: \
https://www.cc.ntust.edu.tw/p/404-1050-91023.php?Lang=zh-tw \
本教學目的是提供說明，讓大家都能架設自己的VPN，**不需透過學校的官方VPN來進行連線** \
免責聲明 :
```
* 本教學不提供任何VPN服務，該方法可能違反學校規範，並且不保證永久有效。
* 強烈譴責任何用於違法/違規之用途，使用者應對其行為負全部責任，作者不承擔任何責任。
* 任何使用的人士，即表示已閱讀、理解並同意以上免責聲明。
```
使用前請詳閱計算機中心之使用規範，確保你沒有違反! \
https://www.cc.ntust.edu.tw/p/412-1050-8537.php?Lang=zh-tw
## 你需要有
* 一台位於校內網路的電腦 (linux/windows/macos...)
* 一台校外的電腦 (linux/windows/macos...)
* 一個zerotier帳號 https://www.zerotier.com/  \
(或者使用具有公共ip之VPS進行異地組網，本文為了方便與金錢考量，使用該服務)
## 名詞解釋
* VPN : 虛擬私人網路，讓你可以透過學校的電腦存取校內網路
* wireguard : 免費開源的VPN伺服器軟體 [wiki](https://zh.wikipedia.org/zh-tw/WireGuard) 
* zerotier : 異地組網服務，用於讓學校電腦可以透過區網訪問，免費版就很夠用了 \
(若是不信任他們的官方伺服器，也可以自己架設[ZeroTierOne](https://github.com/zerotier/ZeroTierOne))
* wg-easy : Wireguard的Docker容器，附帶管理用的webui [Link](https://github.com/wg-easy/wg-easy)
## 架構圖
![image](imgs/architecture.png)
## 加入zerotier網路
此步驟在校外/校內電腦都要進行

### 創建network
註冊並登入zerotier.com，進入 https://my.zerotier.com/ \
按下Create A Network，點擊進入，往下滑，選擇你要使用的區網網段 \
本文預設使用`172.28.*.*` \
![image](imgs/assign_local_network_range.png) \
然後將16碼的network id存起來，之後會用到 \
![image](imgs/network_id.png)

### Ubuntu安裝zerotier
```
sudo apt install curl
curl -s https://install.zerotier.com | sudo bash
sudo zerotier-cli join 替換成你的network_id
```

### Windows安裝zerotier
進入下載頁面，https://www.zerotier.com/download/ \
安裝windows版的exe，在右下角的system tray中找到它，點擊右鍵，Join New Network，輸入network_id \
![image](imgs/add_network_id.png)

### 認證裝置
回到zerotier控制頁面，https://my.zerotier.com/ \
找到新加入的裝置，取個名字，設定一個ip (建議從1開始往上加)，然後再將左邊的Auth勾上
![image](imgs/add_device.png) \
本文預設將校內電腦的ip設為`172.28.0.1`，校外電腦的ip設為`172.28.0.2`

## 架設wireguard vpn伺服器
此步驟在校內電腦進行

### 修改`docker-compose.yaml`
```
# 把本專案clone下來
git clone https://github.com/frakw/ntust-wg-easy.git
cd ntust-wg-easy
```
開啟`docker-compose.yaml`根據你前面在zerotier的配置進行修改
* WG_HOST : 設定成你校內電腦的zerotier ip，本文預設是`172.28.0.1`
* WG_DEFAULT_ADDRESS : VPN的區網網段，本文預設使用`10.8.0.x`
* WG_ALLOWED_IPS : 這邊設定VPN Client可以存取的網段，我們將台科的固定ip(140.118)開放出來，VPN的區網網段也開放出來
* PASSWORD_HASH : wg-easy的一個簡易網頁管理面板的密碼(經過雜湊後)，因為我們的VPN是架在區網，所以不一定需要使用
> 密碼雜湊方法:
> * docker run ghcr.io/wg-easy/wg-easy wgpw 你要的密碼
> * *重要** 把所有$改為$$, EX: $kw$. => $$kw$$.

![image](imgs/modify_docker-compose.yaml.png)

### Ubuntu
```
# 安裝docker, docker-compose，已安裝者可跳過
curl -sSL https://get.docker.com/ | CHANNEL=stable bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```


```
#啟動server
sudo docker compose up -d
```
### Windows
安裝wsl2，已安裝者可跳過
```
wsl --install
```
重新開機，然後安裝Docker Desktop on Windows : \
https://docs.docker.com/desktop/install/windows-install/
```
#啟動server
docker compose up -d
```
> 註: 視情況可能需要關閉windows防火牆才能存取
## 校外電腦連線
### 下載`.conf`檔
進入 http://172.28.0.1:51821/ ，ip是你校內電腦的zerotier區網ip，port是wireguard預設的`51821` \
![image](imgs/wireguard_web.png)
點擊New Client，輸入名字，下載.conf檔
![image](imgs/download_conf.png)
### Ubuntu
```
sudo apt install wireguard resolvconf
sudo mv <conf檔> /etc/wireguard/ntust-wg-easy.conf
wg-quick up ntust-wg-easy.conf
#設定開機後自動啟動，不需要可跳過
systemctl enable wg-quick@ntust-wg-easy
```
### Windows
進入 https://www.wireguard.com/install/ 並安裝軟體 \
點擊 Add Tunnel，選擇剛才下載的.conf檔 \
![image](imgs/windows_wireguard.png) \
點擊Activate按鈕 \
![image](imgs/windows_wireguard_add_conf.png)
### 測試是否成功
進入 https://software.ntust.edu.tw/websoftware/login.aspx ，如果可以成功進去，就是有了 \
![image](imgs/software_ntust.png)
```
#使用你在校內的電腦ip
ping 140.118.x.x
```
如何沒有ping成功可以過一陣子在試，如果還是不行就把server與client都重開看看
```
docker restart ntust-wg-easy
```
然後有個奇怪的現象是，偶爾需要開啟web管理頁面後才能正常連線，我也不知道問題點在哪

---
使用openspeedtest測試的結果，操作一些工作站或傳檔案應該是蠻夠的了 \
![image](imgs/openspeedtest.png)