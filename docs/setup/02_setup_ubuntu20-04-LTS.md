## OS Setup

### UbuntuServer20.04.2LTS Setup

1. GeeekPiケースの組み立て & 結線 & 起動確認
1. [Raspberry Pi Imager](https://www.raspberrypi.org/software/) でSDカードにUbuntuServer20.04.2LTS(64bit) をインストール
1. Raspberry Pi 4へSDカードを挿入し起動
1. wifi設定
   ```
   sudo vim /etc/netplan/50-cloud-init.yaml

   # config check
   sudo netplan --debug try
   sudo netplan --debug generate

   # 適用
   sudo netplan --debug apply
   ```

    <details><summary>/etc/netplan/50-cloud-init.yaml (masterの場合)</summary>

       ```
       network:
         ethernets:
             eth0:
                 dhcp4: true
                 optional: true
         version: 2
         wifis:
           wlan0:
             optional: true
             dhcp4: false
             addresses:
             - 192.168.10.50/24
             gateway4: 192.168.10.1
             nameservers:
               addresses:
               - 8.8.8.8
               - 8.8.4.4
               search: []
             access-points:
               "<SSID名>":
                 password: "<パスワード>"

       ```

    </details>

1. package更新
   ```
   sudo apt update
   sudo apt upgrade -y
   ```
1. 日本語キーボードに変更し再起動
   ```
   sudo dpkg-reconfigure keyboard-configuration
   sudo reboot
   ```
     - `Generic 105-key (Intl) PC` を選択
     - `Japanese` を選択
     - `Japanese` を選択
     - `The default for the keyboard layout` を選択
     - `No compose key` を選択
1. LOCALE
   ```
   sudo apt install -y language-pack-ja
   sudo update-locale LANG=ja_JP.UTF-8
   ```
1. ホスト名 e.g. `k8s-master`
   ```
   name=k8s-master
   echo ${name} | sudo tee /etc/hostname
   sudo sed -i -e 's/127.0.1.1.*/127.0.1.1\t'$name'/' /etc/hosts
   ```

