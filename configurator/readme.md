---
noteId: "7b3178004f7a11f194a2c3b1eecd91b7"
tags: []

---

# configurator

* https://github.com/betaflight/betaflight-configurator

```
sudo apt update
wget https://github.com/betaflight/betaflight-configurator/releases/download/10.10.0/betaflight-configurator_10.10.0_amd64.deb

wget http://archive.ubuntu.com/ubuntu/pool/universe/g/gconf/libgconf-2-4_3.2.6-4ubuntu1_amd64.deb
wget http://archive.ubuntu.com/ubuntu/pool/universe/g/gconf/gconf2-common_3.2.6-4ubuntu1_all.deb
sudo dpkg -i gconf2-common_3.2.6-4ubuntu1_all.deb
sudo dpkg -i libgconf-2-4_3.2.6-4ubuntu1_amd64.deb
sudo dpkg -i betaflight-configurator_10.10.0_amd64.deb
sudo apt-get -f install


```


