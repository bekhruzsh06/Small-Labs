# Samba Lab


## **1. Setting Up a Server**

## **1. Install Samba on Ubuntu Server**

```bash
sudo apt update && sudo apt install samba
```

### **2. Create a Shared Directory**

```bash
mkdir /srv/sambashare/
sudo chmod 777 /srv/sambashare
```

### **3. Configure the Share**

```bash
sudo nano /etc/samba/smb.conf
```


Add this block of text to the very bottom of the file:

```yaml
[sambashare]
comment = Ubuntu File Share
path = /srv/sambashare
read only = no
browsable = yes
guest ok = yes
```


### **4. Restart Service**

```bash
sudo systemctl restart smbd
```


## **2. Accessing Share**

### **1. Via Linux**

#### **1. Mount**

```bash
sudo apt install cifs
```

```bash
mkdir /mnt/sambashare_mnt
mount -t cifs //Ubuntu-IP/sambashare /mnt/sambashare_mnt
```

### **2. Access via smbclient**

```bash
smbclient -N "//Ubuntu-IP/sambashare"
```


### **2. Via Windows**

Via File Explorer-> Подключить сетевой диск
