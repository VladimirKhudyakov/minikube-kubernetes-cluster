# Пошаговая инструкция по развертыванию среды кластера minikube, кластера minikube в VirtualBox на примере тестового проекта

 **Требования:**
 1. Хост машина: Ubuntu 22.04 LTS или аналог;
 2. Доступ к сети интернет с VirtualBox (необходим на весь период развертывания).


> [!NOTE]
> Все команды, указанные в инструкции, выполняются в терминале установленной на шаге 1 виртуальной машины `VirtualBox`, за исключением случаем, когда явно указано где их выполнять.


## Шаг 1: установка VirtualBox (версия 7.1.6).

 На хосте (ПК):

 ```bash
 sudo apt install curl wget gnupg2 lsb-release -y
 curl -fsSL https://www.virtualbox.org/download/oracle_vbox_2016.asc|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/vbox.gpg
 curl -fsSL https://www.virtualbox.org/download/oracle_vbox.asc|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/oracle_vbox.gpg
 
 echo "deb [arch=amd64] http://download.virtualbox.org/virtualbox/debian $(lsb_release -sc) contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list

 sudo apt update
 sudo apt install -y linux-headers-$(uname -r) dkms
 sudo apt install virtualbox-7.1 -y
 sudo usermod -aG vboxusers $USER
 newgrp vboxusers
 ```
 

## Шаг 2: создание виртуальной машины (VirtualBox), с ОС Ubuntu Server 24.04.2 LTS.

  При создании VirtualBox:
  
   **а)** выбрать размер storage: не менее 13GB.
    
   **б)** в разделе `Tools`, в `VirtualBox Manager`, во вкладке `Host-only Networks`, создать сеть c ip адресом `192.168.57.1`
   и маской подсети: `255.255.255.0`, c отключенным `DHCP`. 
    
   **в)** Создать два сетевых адаптера. Первый с типом подключения к сети: `NAT`.
   Второй, с типом подключения к сети: `Host-only`. В поле `name`, выбрать имя сетевого интерфейса сети `Host-only Networks`,
   который был создан на шаге **"б)"**.
    
  
  При установке, выбрать минимальную установку (docker не устанавливать).
  Размета диска: один раздел на весь диск, без использования `LVM` и шифрования.
  
## Шаг 3: добавление созданного при установке ОС пользователя в группу sudo.

 ```bash
 usermod -aG sudo <username>
 ```
  
## Шаг 4: базовая настройка firewall (ufw).

 ```bash
 sudo ufw default deny
 sudo ufw enable 
 ```

## Шаг 5: установка и настройка openssh-server.

 ```bash
 sudo apt update
 sudo apt upgrade
 sudo apt install openssh-server -y
 sudo systemctl enable --now ssh.service
 sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.factory-defaults
 ```
 
 ```bash
 sudo nano /etc/ssh/sshd_config
 ```
 
 или
 
 ```bash
 sudo vi /etc/ssh/sshd_config
 ```
 
 Изменяем следующие параметры:
  
  - Port 11111;
  - AddressFamily inet;
  - ListenAddress 192.168.57.77;
  - PermitRootLogin no;
  - PasswordAuthentication yes
 
 
 ```bash
 sudo sshd -t -f /etc/ssh/sshd_config
 sudo systemctl restart ssh.service
 sudo ufw limit proto tcp from 192.168.57.0/24 to 192.168.57.77 port 11111
 ```
 
## Шаг 6: настройка сети

 1. Открываем файл `50-cloud-init.yaml` в директории `/etc/netplan`:
  
 ```bash
 sudo nano /etc/netplan/50-cloud-init.yaml
 ```
 (если файла `50-cloud-init.yaml` нет, должен быть аналогичный, в формате .yaml, в этой же директории).
  
  
 2. Полностью очищаем его и вставляем следующие настройки:
 ```
 network:
 version: 2
 ethernets:
   enp0s3:
       dhcp4: true
       dhcp6: false
   enp0s8:
       dhcp4: false
       dhcp6: false
       addresses:
           - 192.168.57.77/24
       routes:
           - to: 192.168.57.0/24
             via: 192.168.57.1
             on-link: true
 ```
 , где:
   - `enp0s3` - сетевой интерфейс с типом подключения к сети: `NAT` (смотри шаг 2);
   - `enp0s8` - сетевой интерфейс с типом подключения сети: `Host-only`.
   
 *Имена интерфейсов в данном руководстве и на вашей `VirtualBox`, могут отличаться.*
 *При их различии, необходимо указать актуальные имена в вашей `VirtualBox`).*
    
 Файл с данными настройками (`50-cloud-init.yaml`), есть в папке файлов проекта `project_files`, в подкаталоге `virtualbox_server_files`.
  
 3. Применяем настройки.
  
 ```bash
 sudo netplan apply
 ```
  
## Шаг 7: копирование файлов проекта с хоста (ПК) на VirtualBox.

 **На хосте (ПК):**

 ```bash
 scp -r -P 11111 <path_dir>/project_files/virtualbox_server_files/home_directory work@192.168.57.77:/home/work/Downloads
 ```
 
 **Заходим в VirtualBox по ssh и в ее терминале выполняем команды:**
 ```bash
 cd $HOME
 cp -a -T /home/work/Downloads/home_directory /home/work
 rm -r /home/work/Downloads
 ```


## Шаг 8: установка необходимых утилит

 ```bash
 sudo apt update
 sudo apt install cat nano vim wget -y
 ```
 
## Шаг 9: установка Docker Engine

 ```bash
 # Add Docker's official GPG key:
 sudo apt update
 sudo apt install apt-transport-https ca-certificates curl gnupg
 sudo install -m 0755 -d /etc/apt/keyrings
 sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
 sudo chmod a+r /etc/apt/keyrings/docker.asc

 # Add the repository to Apt sources:
 echo \
   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
   $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 sudo apt-get update 
 
 sudo apt install docker-ce
 sudo systemctl enable --now docker
 
 # check if docker is running
 sudo systemctl status docker
 
 # manage Docker as a non-root user
 sudo groupadd docker
 sudo usermod -aG docker $USER && newgrp docker
 ``` 
       
## Шаг 10: установка kubectl

 ```bash
 sudo apt-get update
 
 curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
 sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

 # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
 echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
 sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly
 
 sudo apt-get update
 sudo apt-get install -y kubectl
 ```  

## Шаг 11: загрузка необходимых Docker образов из удаленного реестра (Docker Hub)

### nginx образ

 ```bash
 docker image pull nginx
 ```
 
### nginx-prometheus-exporter образ

  ```bash
  docker image pull nginx/nginx-prometheus-exporter
  ```


## Шаг 12: построение кастомного образа Docker nginx

 1. Необходимо перейти в домашнюю директорию, в которую, предварительно, загружены файлы проекта (из каталога `project_files`).
 2. Перейти в каталог `docker_nginx`.
 3. Выполнить команду:

 ```bash
 docker build -t custom-nginx .
 ```

 
## Шаг 13: установка minikube 

 ```bash
 curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
 
 sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
 ```

 
## Шаг 14: запуск, инициализация и доустановка необходимых компонентов minikube 

 ```bash
 minikube start --driver=docker
 ```
 
### включение поддержки и установка дополнения 'ingress'
 
 ```bash
 minikube addons enable ingress
 ```

## Шаг 15: загрузка необходимых Docker образов из локального репозитория ОС в кластер minikube

 ```bash
 minikube image load custom-nginx
 minikube image load nginx/nginx-prometheus-exporter
 ```
  
## Шаг 16: создание deployments, services, ingresses 

 Cоздание deployments, services, ingresses нашего проекта, 
 из файлов манифестов (.yaml) в каталоге `.web-nginx-mk/` (предварительно загруженные), в домашней директории ОС:

 ```bash
 kubectl apply -f $HOME/.web-nginx-mk/deployments/web-nginx-mk.yaml
 kubectl apply -f $HOME/.web-nginx-mk/services/web-nginx-mk.yaml
 kubectl apply -f $HOME/.web-nginx-mk/ingresses/web-nginx-mk.yaml
 ```
 
 
## Шаг 17: установка Prometheus и Grafana в minikube используя Helm

### установка Helm

 ```bash
 curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
 chmod 700 get_helm.sh
 ./get_helm.sh
 rm ./get_helm.sh
 ```

### добавление необходимых репозиториев

 ```bash
 helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
 helm repo add grafana https://grafana.github.io/helm-charts
 helm repo update
 ```
 
### установка prometheus

 ```bash
 helm install prometheus prometheus-community/prometheus -f $HOME/.prometheus/values.yaml
 ```

### установка grafana

 ```bash
 helm install grafana grafana/grafana
 ```

## Шаг 18: создание ingress для Grafana

  Создание ingress для Grafana, из файла манифеста (.yaml), в каталоге `.grafana/` (предварительно загруженный), из домашней директории ОС `VirtualBox`:

 ```bash
 kubectl apply -f $HOME/.grafana/ingresses/grafana.yaml
 ```


## Шаг 19: Настраиваем фаейрвол (iptables)
  
 Настраиваем фаейрвол на виртуальной машине `VirtualBox` (c установленным кластером `minikube`), 
 для доступа к `nginx` и `grafana` (через curl или веб браузер) с хоста (ПК), на
 котором она установлена (данные правила необходимы только для указанного доступа).
  
 Доступ из вне кластера к `grafana` или к `nginx` сервису, из самой виртуальной машины `VirtualBox`, существует и без них. 
 
 **(Эти правила временные и после перезагрузки виртуальной машины `VirtualBox` не сохраняются)**

 ```bash
 sudo iptables -t nat -I PREROUTING 1 -p tcp -i enp0s8 --dport 80 -j DNAT --to-destination 192.168.49.2:80
 sudo iptables -I FORWARD 1 -i enp0s8 -o br-7f747a2e96dd -j ACCEPT
 ```
 
 , где:
   - `enp0s8` - сетевой интерфейс в виртуальной машине `VirtualBox`, через который подключено соединение типа мост (bridge) с хостом (ПК, на котором установлена виртуальная машина (VirtualBox)), имеющий статический ip адрес (в данном примере: `192.168.57.77`).
   - `br-7f747a2e96dd` - сетевой интерфейс - мост (bridge) в виртуальной машине `VirtualBox`, созданный кластером `minikube`, после его развертывания. 
   Имеет ip адрес из одной подсети с ip адресом назначеного `Ingress`-ам `grafana` и `nginx` сервису (в данном примере, ip адрес сетевого интерфейса `br-7f747a2e96dd`: `192.168.49.1`).

***ВАЖНО!!!*** Указанные сетевые интерфейсы могут иметь другие имена. Поэтому, при настройке данных правил, необходимо указать актуальные имена соответствующих сетевых интерфейсов 
в вашей ОС виртуальной машины `VirtualBox`.
   
Команда в терминале (в виртуальной машине `VirtualBox`): ```bash ip a```, покажет имена сетевых интерфейсов и другую информацию.
      
Добавляем настройку в параметры ядра Linux для forward трафика. Открываем файл `sysctl.conf` 
```bash
sudo nano /etc/sysctl.conf
```
Устанавливаем параметр `net.ipv4.ip_forward`:
```bash
net.ipv4.ip_forward=1
```
Сохраняем изменения в файле и закрываем его. Активируем все настройки ядра.
```bash
sudo sysctl -p
```
Перезапустим файервол `ufw` для обновления настроек.
```bash
sudo systemctl restart ufw
```

## Шаг 20: настройка доступа к grafana или nginx сервису из VirtualBox

 Необходимо добавить в файл /etc/hosts на виртуальной машине VirtualBox (c установленным кластером minikube), в которой будет выполняться внешний запрос (curl или веб браузер) 
 к grafana  или nginx сервису.

 ```
 192.168.49.2 web-nginx-mk.localcluster.mk
 192.168.49.2 grafana.localcluster.mk
 ```
 
 , где `192.168.49.2` - ip адрес назначенный `ingress` (посмотреть точный ip адрес, можно выполнив следующую команду: ```kubectl get ingress```, в терминале виртуальной машины `VirtualBox`, на которой установлен кластер `minikube`).


## Шаг 21: настройка доступа к grafana или nginx сервису с хоста (ПК)

 Для выполнения внешнего запроса (curl или веб браузер) к grafana или nginx сервису, с хоста (ПК), с установленной на нем виртуальной машины VirtualBox (c установленным кластером
 minikube), необходимо добавить в файл /etc/hosts, на хосте (ПК), указанные строки (данные правила необходимы только для указанного доступа):

 ```
 192.168.57.77 web-nginx-mk.localcluster.mk 
 192.168.57.77 grafana.localcluster.mk
 ```

 , где `192.168.57.77` - статический IP адрес, назначенный одному из сетевых интерфесов виртуальной машины (`VirtualBox`) для соединения с хостом (ПК), на котором она установлена
 (тип соединение с хостом (ПК):  мост (bridge)).
 
 
## Шаг 22: настройка визуализации мониторинга в Grafana 

 Действия выполняются на хосте (ПК), на котором установлена VirtualBox (c установленным кластером `minikube`).
 
 1. Открываем web-интерфейс Grafana в web-браузере (ссылка: http://grafana.localcluster.mk)
 
 2. Для получения пароля, в терминале виртуальной машины `VirtualBox` (c установленным кластером `minikube`) вводим команду:
  ```kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo```
  (вывод команды - является паролем).
   
 3. Вводим в web-интерфейсе пароль `admin` и пароль (получен в пункте 2).

 4. После входа, в основном меню, с левой стороны, выбираем раздел `Connections`, в котором, после перехода на новую страницу, выбираем из источников данных `Data Source`
 источник `Prometheus`.
 
 5. В следующем интерфейсе, нажимаем кнопку `Add new data source` (в верхнем правом углу страницы).
 6. На новой странице, в модуле `Connection`, в поле `Prometheus server URL`, вводим следующий URL: 'http://prometheus-server.default.svc.cluster.local'.
 7. В конце страницы, нажимаем на кнопку: `Save & test`. При успешном тестировании соеднинения с источником данных, появится сообщение: *"Successfully queried the Prometheus API"*.
    (при ошибке, система `Grafaba` выведет соответствующее сообщение). 
 8. Далее, в панели основного меню, с левой стороны, выбираем раздел `Dashboards`.
 9. На следующей странице, нажимаем кнопку `New` (в верхнем правом углу страницы) и в выпадающем списке выбираем `Import`. 
 10. На новой странице, в поле `Import via dashboard JSON model`, копируем полное содержимое файла `11199_rev1.json` ( `project_files/grafana_dashbord`),
     который был предварительно загружен вместе с остальными файлами проекта на хост (ПК).
 11. Далее, нажимаем кнопку `Load`.
 12. При удачной загрузке данных, на этой же страницы, появится поле `Prometheus`, в котором необходимо выбрать источник данных `Prometheus` из выпадающего списка.
 13. Нажимаем кнопку `Import`. При положительном результате, появится дашборд с базовыми метриками `Nginx`.
  


## Доступ к развернутым ресурсам:

 *доступ к серверу nginx (curl или веб браузер) можно получить по ссылке:* `http://web-nginx-mk.localcluster.mk`
 
 *доступ к серверу nginx (curl или веб браузер) можно получить по ссылке:* `http://grafana.localcluster.mk`



