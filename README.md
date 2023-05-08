# kubernetes-cluster

Автоматизация по запуску кластера Kubernetes
---------------
Разворачивать будем kubernetes с помощью kubespray, это набор ansible ролей для разворачивания кластера на серверах. Сначала настроим серверы, потом скачаем из репозитория и настроим  Kubespray.

За основу используется ОС CentOS 7
Запущен один сервер управления
Запущены три control сервера
Запущены три сервера worker nodes
На каждом сервере обновлены пакеты до последнх версий:
```
yum update
```
Был создан ssh ключ:
```
ssh-keygen
```
И скопирован на каждый сервер:
```
ssh-copy-id root@server_ip_address
```

Настраиваем DNS сервер:
```
yum install bind bind-utils
```
В параметрах `/etc/named.conf` указываем настройки:
```
options {
        listen-on port 53 { 127.0.0.1; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { localhost; };
        recursion yes;

        dnssec-enable yes;
        dnssec-validation yes;

        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { none; };
};

zone "boris.local" IN {
        type master;
        file "boris.local";
        allow-update { none; };
};

zone "0.168.192.in-addr.arpa" IN {
        type master;
        file "247";
        allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```
Создаём файл `/var/named/boris.local` с зоной прямого преобразования "boris.local":
```
$TTL 86400
@ IN SOA master.boris.local. admin.boris.ru. (
                                            2021012100 ;Serial
                                            3600 ;Refresh
                                            1800 ;Retry
                                            604800 ;Expire
                                            86400 ;Minimum TTL
)

@ IN NS master

master          IN      A       192.168.247.147
control1        IN      A       192.168.247.148
control2        IN      A       192.168.247.149
control3        IN      A       192.168.247.150

worker1         IN      A       192.168.247.151
worker2         IN      A       192.168.247.152
worker3         IN      A       192.168.247.153
```
и зоной обратного преобразования в файле `/var/named/247`:
```
$TTL 86400
@ IN SOA master.boris.local. admin.boris.ru. (
                                            2021012100 ;Serial
                                            3600 ;Refresh
                                            1800 ;Retry
                                            604800 ;Expire
                                            86400 ;Minimum TTL
)
@ IN NS master.boris.local.

147 IN PTR master.boris.local.

148     IN      PTR     control1.boris.local.
149     IN      PTR     control2.boris.local.
150     IN      PTR     control3.boris.local.

151     IN      PTR     worker1.boris.local.
152     IN      PTR     worker2.boris.local.
153     IN      PTR     worker3.boris.local.
```
Проверяем конфиг на ошибки:
```
named-checkconf
```
Запускаем named, проверяем статус, добавляем в автозагрузку:
```
systemctl start named
systemctl status named
systemctl enable named
```
Устанавливаем pip3:
```
yum install epel-release
yum install python3-pip
```
для успешного развертывания на всех серверах были отключены службы:
- SELinux (средства контроля доступа к файловой системе):
- Firewall
- Swap

Можно настроить вручную, а можно воспользоваться ролью `configuration-servers/prepare-hosts.yml`
Необходимо только поменяить ip адреса на ip адреса ваших серверов в файле `hosts.txt`
```
ansible-playbook prepare-hosts.yml
```
Скачаем Kubespray из репозитория:
```
git clone https://github.com/kubernetes-sigs/kubespray
```
устанавливаем зависимости из директории `kubespray`:
```
pip3 install -r requirements.txt
```
Переходим к настройке kubespray:
Скопируем пример директории `/kubespray/inventory/sample` в ту же директорию и назовём её cluster
Приводим файл `cluster/inventory.ini` к виду:
```
[all]
control1.boris.local ansible_host=192.168.247.148
control2.boris.local ansible_host=192.168.247.149
control3.boris.local ansible_host=192.168.247.150
worker1.boris.local ansible_host=192.168.247.151
worker2.boris.local ansible_host=192.168.247.152
worker3.boris.local ansible_host=192.168.247.153

[kube-master]
control1.boris.local
control2.boris.local
control3.boris.local

[etcd]
control1.boris.local
control2.boris.local
control3.boris.local

[kube-node]
worker1.boris.local
worker2.boris.local
worker2.boris.local

[calico-rr]

[k8s-cluster:children]
kube-master
kube-node
calico-rr
```
Далее переходим к настройке файла `group_vars/k8s-cluster/k8s-cluster.yml`:
выбираем версию кластера кубернетес:
```
kube_version: v1.20.2
```
выбираем драйвер сети кластера:
```
kube_network_plugin: calico
```
диапазон адресов для сервисов кластера:
```
kube_service_addresses: 10.233.0.0/18
```
диапазон адресов для подов кластера:
```
kube_pods_subnet: 10.233.64.0/18
```
размер подсети подов на ноде кластера:
```
kube_network_node_prefix: 24
```
Порт API сервера кластера:
```
kube_apiserver_port: 6443
```
режим iptables не ставим:
```
kube_proxy_mode: ipvs
```
задаем имя кластера, используется в качестве корневого домена во внутреннем DNS сервере:
```
cluster_name: cluster.local
```
включаем кеширующие DNS сервера на каждой ноде кластера:
```
enable_nodelocaldns: true
```
IP адрес кешируюшего DNS сервера на ноде:
```
nodelocaldns_ip: 169.254.25.10
```
определяем систему контейнеризации:
```
container_manager: containerd
```
политика загрузки образов системных контейнеров кластера:
```
k8s_image_pull_policy: IfNotPresent
```
зарезервированная за Linux системой (приложениями) память:
```
system_memory_reserved: 512Mi 
```
зарезервированное за Linux системой (приложениями) время процессора:
```
system_cpu_reserved: 500m
```
Автоматический перевыпуск сертификатов для кубернетес control plane. Без необходимости увеличения версии кластера:
```
force_certificate_regeneration: true
```

Так же необходимо в файле `group_vars/k8s-cluster/k8s-net-calico.yml` определить когда использовать IP in IP режим:
```
calico_ipip_mode: 'CrossSubnet'
```
В файле `group_vars/all/all.yml`:
разрешаем установку и etcd средствами kubeadm:
```
etcd_kubeadm_enabled: true
```
Доступ к k8s API через loopback интерфейс ноды кластера (значение по умолчанию):
```
loadbalancer_apiserver_type: nginx
```
Настройки сделаны, можно запускать плейбук для размертывания kubernetes:
```
ansible-playbook -i inventory/cluster/inventory.ini cluster.yml
```
После установки проверим работу нодов, переключимся на control1 сервер и введём команду:
```
kubectl get nodes
```
В выводе команды должно отображаться наши 6 серверов

