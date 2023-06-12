Задание 1
Возьмите за основу решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке».

Теперь вместо одной виртуальной машины сделайте terraform playbook, который:
создаст 2 идентичные виртуальные машины. Используйте аргумент count для создания таких ресурсов;
создаст таргет-группу. Поместите в неё созданные на шаге 1 виртуальные машины;
создаст сетевой балансировщик нагрузки, который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.
Рекомендуем изучить документацию сетевого балансировщика нагрузки для того, чтобы было понятно, что вы сделали.

Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

Перейдите в веб-консоль Yandex Cloud и убедитесь, что:

созданный балансировщик находится в статусе Active,
обе виртуальные машины в целевой группе находятся в состоянии healthy.
Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.
В качестве результата пришлите:

1. Terraform Playbook.
Ч.1
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/91385b12-8ac0-4834-84d6-87333f79af7d)
Ч.2
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/67a0dfd6-4e6b-4e13-98e8-2ac85af1bff5)

В текстовом формате:

terraform {
  required_version = "= 1.4.5"

  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = "= 0.73"
    }
  }
}

provider "yandex" {
  token     = "y0_AgAAAAA_PcoqAATuwQAAAADhiQyj1ODEvU8yQ-SzT5d3Uw1pzODQqpo"
  cloud_id  = "cloud-a-n-schelokov"
  folder_id = "b1giuo998cutbq8j87ju"
  zone      = "ru-central1-b"
}

resource "yandex_compute_instance" "vm" {
  count = 2
  name  = "vm${count.index}"

  resources {
    cores  = 2
    memory = 2
  }

placement_policy {
    placement_group_id = "${yandex_compute_placement_group.group1.id}"
  }

  boot_disk {
    initialize_params {
      image_id = "fd89dg1rq7uqslc6eigm"
      size = 5
    }
  }
  network_interface {
    subnet_id = "${yandex_vpc_subnet.subnet-1.id}"
    nat       = true
  }
  metadata = {
    user-data = "${file("/usr/local/bin/meta.yml")}"
  }
}
resource "yandex_compute_placement_group" "group1" {
  name = "test"
}
resource "yandex_vpc_network" "network-1" {
  name = "network-1"
}
resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet-1"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.network-1.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_lb_network_load_balancer" "test" {
  name = "lb-test"
  listener {
    name = "test"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }
  attached_target_group {
    target_group_id = yandex_lb_target_group.test.id
    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

resource "yandex_lb_target_group" "test" {
  name = "test"
  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = "${yandex_compute_instance.vm[0].network_interface.0.ip_address}"
  }
  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = "${yandex_compute_instance.vm[1].network_interface.0.ip_address}"
  }
}


2. Скриншот статуса балансировщика и целевой группы.
Целевая группа:
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/d6c52707-0167-4d01-9ede-e445d19cba28)
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/877f7818-faef-4931-93e0-6aa8db299761)
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/6bd60971-2065-4eff-b543-7ff06d1c8318)

Статус баланщировщика:
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/66864eb6-1a89-4b4a-89aa-2c28a31df5ba)
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/604a0224-2589-475e-b74a-6cdd33358d53)

3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.
Nginx развернут по playbook
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/1a81bc6b-4e1e-47b8-949b-6bbed0f43a6e)
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/582085dd-124c-4d3f-93eb-d950eee17eac)
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/2a970a43-6080-4e21-8cee-83568654af29)
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/dd86db33-85f6-4209-96b0-4f682fa6d40a)
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/99784b98-b1f2-417a-b332-cc4683f7cf49)
отключил 158.160.68.78
![image](https://github.com/AlexanderSchelokov/Fault-tolerance-in-the-cloud-hw/assets/121572590/ea2da434-358d-4884-a467-65769669cc35)




