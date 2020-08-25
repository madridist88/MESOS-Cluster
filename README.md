 - Что это такое?

Это скрипты сборки кластера Мясосферы на некотором количестве виртуальных машин. И скрипт добавления новой ноды в Мясосферный кластер.

 - Что для этого нужно?
 
Нужны ноды для Мясосферы:
        bootstrap - 1 штука
        master - минимум 1 штука, в документации требуют 3, дабы обеспечить кворум в zookeeprer.
        node - чем больше тем круче. Вычислительные ноды кластера.
На нодах должен быть установлен CentOS и везеде заведен один и тот же пользователь с одним и там же паролем. На bootstrap ноде должен быть установлен git и ansible
На текущем этапе ноды создаются в вмваре из темплейта.

 - Как этим пользоваться?
 
    1. Идем на ноду которую мы выбрали как bootstrap. Делаемся там пользователем от которго будем собирать кластер. У пользователя должны быть рутовые права. В примере это сам root
    2. Клонируем репозиторий на ноду
   
        
    3. Переходим в директорию и правим инвентарный файлик:
    
        cd dcos
        vim ansible/inventories/hosts.ini
        
        В файлик записываем имена хостов и их адреса. Хосты записываются в соответствующие группы. Это определяет их роль в кластере.
        И задаются переменные: Имя будущего кластера, пользователь и пароль для подключени к нодам и переменные для настройки сервисов..
        Напрмер:
        
            [node:children]
            node-public
            node-private

            [node-public]
            isdk-rc-node01 ansible_ssh_host=10.5.0.101

            [node-private]
            isdk-rc-node02 ansible_ssh_host=10.5.0.102
            isdk-rc-node03 ansible_ssh_host=10.5.0.103
            isdk-rc-node04 ansible_ssh_host=10.5.0.104

            [master]
            isdk-rc-master01 ansible_ssh_host=10.5.0.100    nfs_master=1

            [bootstrap]
            isdk-rc-bootstrap ansible_connection=local

            [all:vars]
            ansible_user=admin
            ansible_ssh_pass=qWerty123!
            cluster_name=isdk-rc
            cluster_domain=onelya.ru
            cluster_nameserver=10.5.0.20

            docker_registry_user=admin
            docker_registry_password=qWerty123
        
        В переменных задаются следующие значения:
        ansible_user - пользователь для подключения к нодам
        ansible_ssh_pass - пароль пользователя
        cluster_name - имя кластера, который будет создаваться
        cluster_domain - домен на котором будет работать кластер. 
        cluster_nameserver - ДНС сервер который будет использоваться кластером. Нужен чтобы избежать хождения трафика через внешние сети.
        
    4. После приведения файлика в актуальное состояние выполняем следующие плейбуки по мере необходимости:
    
        ansible-playbook -i ansible/inventories/hosts.ini ansible/playbook/prepare-hosts.yml - подготовка хостов для установки dcos, установка сервосов и Docker
        
        ansible-playbook -i ansible/inventories/hosts.ini ansible/playbook/make-crt.yml - Создание, ПРИ НЕОБХОДИМОСТИ, самоподписанного сертификата
        и добавление его в довененные на хостах. Сертификат генерится для домена заданного переменной cluster_domain
        
        ansible-playbook -i ansible/inventories/hosts.ini ansible/playbook/cluster_setup.yml - Установка кластера Mesos.
        
        ansible-playbook -i ansible/inventories/hosts.ini ansible/playbook/cluster_node_setup.yml - Добавление ноды в кластер.
        При этом нужно предварительно добавить новую ноду в ansible/inventories/hosts.ini
        
        ansible-playbook -i ansible/inventories/hosts.ini ansible/playbook/services_setup.yml - Поднятие сервисов внутри mesos ксластера.
        При это используются сертификаты из директрии ~/ssl/ Если необходимы доверенные, их нужно разместить в эту директорию.
        Или сренерировать самоподписанные плейбуком ansible/playbook/make-crt.yml
        И задать переменные для настройки сервисов. К примеру docker-registry.
        Перед запуском плейбука необходимо установить dcos-cli и выполнить авторизацию в кластере. Например так:
            [ -d /usr/local/bin ] || sudo mkdir -p /usr/bin && 
            curl https://downloads.dcos.io/binaries/cli/linux/x86-64/dcos-1.11/dcos -o dcos && 
            sudo mv dcos /usr/bin && 
            sudo chmod +x /usr/bin/dcos && 
            dcos cluster setup http://10.5.0.100 && 
            dcos

        ansible-playbook -i ansible/inventories/hosts.ini ansible/playbook/make-nfs.yml - Установка NFS. Не доделано.

        

        
