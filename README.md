# Домашнее задание к занятию "Отказоустойчивость в облаке" - `Кошелев Дмитрий Владимирович`


  
---

### Задание 1

# Домашнее задание: Развертывание инфраструктуры в Yandex Cloud с балансировщиком

## Задание 1: Развертывание VPC, двух веб-серверов и сетевого балансировщика

### Описание задания
Создана инфраструктура в Yandex Cloud, включающая:
- Виртуальную сеть (VPC) с подсетью
- Две идентичные виртуальные машины с веб-сервером Nginx
- Целевую группу для балансировщика
- Сетевой балансировщик нагрузки с проверкой состояния (Health Check)

### Выполненные шаги

1. **Создание облачной сети и подсети:**
   - Создана VPC сеть `app-network` с CIDR `10.10.0.0/24`
   - Создана подсеть `app-subnet` в зоне `ru-central1-a`

2. **Создание виртуальных машин:**
   - Созданы две ВМ: `web-server-1` и `web-server-2`
   - Использован образ Ubuntu 22.04 LTS
   - Назначены публичные IP-адреса для доступа из интернета
   - Настроен SSH-доступ через ED25519 ключ

3. **Установка и настройка Nginx:**
   - На обе ВМ установлен веб-сервер Nginx
   - Сервис добавлен в автозагрузку
   - Проверена доступность через публичные IP-адреса

4. **Создание целевой группы:**
   - Создана целевая группа `app-target-group`
   - В группу добавлены обе ВМ с их внутренними IP-адресами
   - Настроена проверка состояния (Health Check) на порт 80

5. **Создание сетевого балансировщика:**
   - Создан внешний балансировщик `app-load-balancer`
   - Настроен обработчик (listener) на порт 80
   - Трафик направляется на целевой порт 80 ВМ
   - Настроена HTTP-проверка состояния по пути `/`

### Terraform Playbook

```hcl
# 1. Конфигурация провайдера
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
      version = ">= 0.47.0"
    }
  }
}

provider "yandex" {
  service_account_key_file = "key.json"
  cloud_id  = "ваш_cloud_id"
  folder_id = "ваш_folder_id"
  zone      = "ru-central1-a"
}

# 2. Создание сети и подсети


```
resource "yandex_vpc_network" "app-network" {
  name = "app-network"
}

resource "yandex_vpc_subnet" "app-subnet" {
  name           = "app-subnet"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.app-network.id
  v4_cidr_blocks = ["10.10.0.0/24"]
}

# 3. Создание двух виртуальных машин с Nginx (через count)
resource "yandex_compute_instance" "web-server" {
  count = 2
  name  = "web-server-${count.index + 1}"
  zone  = "ru-central1-a"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8f6b7d6a2c7f3e8f1a"  # Ubuntu 22.04 LTS
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.app-subnet.id
    nat       = true
  }

  metadata = {
    user-data = <<-EOF
    #cloud-config
    package_update: true
    packages:
      - nginx
    runcmd:
      - systemctl enable nginx
      - systemctl start nginx
    EOF
  }
}

# 4. Создание целевой группы
resource "yandex_lb_target_group" "app-target-group" {
  name = "app-target-group"

  dynamic "target" {
    for_each = yandex_compute_instance.web-server.*
    content {
      subnet_id = yandex_vpc_subnet.app-subnet.id
      address   = target.value.network_interface.0.ip_address
    }
  }
}

# 5. Создание сетевого балансировщика
resource "yandex_lb_network_load_balancer" "app-load-balancer" {
  name = "app-load-balancer"
  type = "external"

  listener {
    name = "http-listener"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.app-target-group.id

    healthcheck {
      name = "http-healthcheck"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

# 6. Вывод информации
output "load_balancer_external_ip" {
  value = yandex_lb_network_load_balancer.app-load-balancer.listener.0.external_address_spec.0.address
  description = "Внешний IP адрес балансировщика"
}

output "vm_external_ips" {
  value = yandex_compute_instance.web-server.*.network_interface.0.nat_ip_address
  description = "Внешние IP адреса ВМ"
}

output "vm_internal_ips" {
  value = yandex_compute_instance.web-server.*.network_interface.0.ip_address
  description = "Внутренние IP адреса ВМ"
}...
....
....
....
....
```

`При необходимости прикрепитe сюда скриншоты
![Задание 1](./img/1.png);
![Задание 1](./img/2.png);
![Задание 1](./img/3.png);
![Задание 1](./img/4.png);
![Задание 1](./img/5.png);

