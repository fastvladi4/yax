[# yax](https://cloud.vk.com/docs/tools-for-using-services/terraform/reference/configuration
и надо на опенстек искать методич

```
sudo apt-get update && sudo apt-get install -y wget unzip
wget https://hashicorp-releases.yandexcloud.net/terraform/1.10.3/terraform_1.10.3_linux_arm64.zip
sudo unzip  terraform_1.10.3_linux_amd64.zip -d /usr/local/bin/
sudo apt-get install -y python3-module-openstackclient python3-module-pip
pip3 install python-octaviaclient python-cinderclient python-novaclient
```
#подключ тераформ
vim ~/.terraformrc

```
provider_installation {
    network_mirror {
        url = "https://terraform-mirror.mcs.mail.ru"
        include = ["registry.terraform.io/*/*"]
    }
    direct {
        exclude = ["registry.terraform.io/*/*"]
    }
}
```
```
    mkdir ~/bin && cd ~/bin
    vim cloud.conf


Помещаем в данный файл следущее содержимое:
    
    # Terraform
    export TF_VAR_OS_AUTH_URL=https://cyber-infra.local.prof:5000/v3
    export TF_VAR_OS_PROJECT_NAME=Project1
    export TF_VAR_OS_USERNAME=user01
    export TF_VAR_OS_PASSWORD=user01P@ssw0rd
    
    # openstacl-cli
    export OS_AUTH_URL=https://cyber-infra.local.prof:5000/v3
    export OS_IDENTITY_API_VERSION=3
    export OS_AUTH_TYPE=password
    export OS_PROJECT_DOMAIN_NAME=Region2025
    export OS_USER_DOMAIN_NAME=Region2025
    export OS_PROJECT_NAME=Project1
    export OS_USERNAME=user01
    export OS_PASSWORD=user01P@ssw0rd
```

```
 source cloud.conf
 openstack --insecure server list
```

```
vim provider.tf

variable "OS_AUTH_URL" {
	type = string
	sensitive = true
}

variable "OS_PROJECT_NAME" {
	type = string
	sensitive = true
}

variable "OS_USERNAME" {
	type = string
	sensitive = true
}

variable "OS_PASSWORD" {
	type = string
	sensitive = true
}
```

```
terraform init
```

#
далее 552.html
#

Все файлы создаются в контексте каталога /home/altlinux/bin, если не сказано иное
Удаляем все ранее созданные ресурсы средствами Terraform для дальнейшего развёртывания средствами одного файла cloudinit.sh:

    terraform destroy
    
    yes
#
cloudinit.sh
```
#!/bin/bash

cd /home/altlinux/bin
source cloud.conf
terraform init
terraform apply -auto-approve
terraform output > /home/altlinux/white.ip
ansible-playbook -i ansible/inventory ansible/wireguard_playbook.yml
ansible-playbook -i ansible/inventory ansible/ssh_playbook.yml

echo "Проверяем доступность созданных инстансов, для каждого инстанса статус должен быть ACTIVE:"
echo ""
openstack --insecure server list

echo "Проверяем доступность созданного балансировщика нагрузки:"
echo ""
openstack --insecure loadbalancer list

echo "Проверяем доступность Web-серверов через балансировщик нагрузки:"
echo ""
openstack --insecure loadbalancer member list HTTP
openstack --insecure loadbalancer member list HTTPS
```

```
chmod +x cloudinit.sh
./cloudinit.sh
```

#

vim ~/py.py
```
import os

def main():
    working_directory = os.path.expanduser("/root")
    file_path = os.path.join(working_directory, "input.txt")

    if os.path.exists(file_path):
        with open(file_path, "r", encoding="utf-8") as file:
            content = file.read()
            print(content)
    else:
        print("Ошибка: файл input.txt не найден в директории /root.")

if __name__ == "__main__":
    main()
```
создаём input.txt и туда рандом текст пишем
#
vim ~/Dockerfile
```
FROM python:3.8-alpine
WORKDIR /root
COPY py.py .
COPY input.txt .
CMD ["python", "py.py"]
```
```
sudo apt-get install -y docker-engine
sudo systemctl enable --now docker.service
sudo docker build -t file-copy-python
```

```
sudo docker run --rm file-copy-python
```

#

c) Развертывание WordPress с использованием Docker Compose


vim ~/wordpress.yml
```
version: '3.1'

services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    networks:
      - wordpress-network
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: mysql:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
    depends_on:
      - mysql
    volumes:
      - wordpress_data:/var/www/html

  mysql:
    image: mysql:5.7
    container_name: mysql
    networks:
      - wordpress-network
    environment:
      MYSQL_ROOT_PASSWORD: toor
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    volumes:
      - mysql_data:/var/lib/mysql

networks:
  wordpress-network:
    driver: bridge
    name: wordpress-network

volumes:
  wordpress_data:
  mysql_data:
  
```
```
sudo apt-get install -y docker-compose-v2
sudo docker compose -f wordpress.yml up -d
```

proverka
```
sudo docker ps
sudo docker network ls
Проверяем доступ к веб-интерфейсу настройки WordPress:

    Обращаясь на Плавающий-IP адрес ControlVM на порт 80;

```
#
Задание:

4) Развертывание базового стека ELK

vim ~/elk.yml
```
version: '3.7'

services:
  elasticsearch:
    image: elasticsearch:7.10.1
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  logstash:
    image: logstash:7.10.1
    container_name: logstash
    depends_on:
      - elasticsearch
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    environment:
      LS_JAVA_OPTS: "-Xms1g -Xmx1g"

  kibana:
    image: kibana:7.10.1
    container_name: kibana
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: "http://elasticsearch:9200"

volumes:
  es_data:
```


vim ~/logstash.conf
```
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
}
```
```
sudo docker compose -f elk.yml up -d
```)
