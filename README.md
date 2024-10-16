## Анализ и планирование

## функциональность текущего монолитного приложения.

**Управление отоплением:**

- Пользователи могут удалённо включать/выключать отопление в своих домах.
- Пользователи могут устанавливать желаемую температуру.
- Система автоматически поддерживает заданную температуру, регулируя подачу тепла.

**Мониторинг температуры:**

- Система получает данные о температуре с датчиков, установленных в домах.
- Пользователи могут просматривать текущую температуру в своих домах через веб-интерфейс.

## Архитектура монолитного приложения

- **Язык программирования:** Java
- **База данных:** PostgreSQL
- **Архитектура:** Монолитная, все компоненты системы (обработка запросов, бизнес-логика, работа с данными) находятся в рамках одного приложения.
- **Взаимодействие:** Синхронное, запросы обрабатываются последовательно.
- **Масштабируемость:** Ограничена, так как монолит сложно масштабировать по частям.
- **Развертывание:** Требует остановки всего приложения.

**домены и границы контекстов**
- управление устройствами 
	- управление отоплением
- мониторинг датчиков
	- мониторинг температуры
- продажи услуг подключения
	- управление пользователями
	- обработка платежей
		- управление транзакциями

```puml
@startuml
!theme plain
set separator none
title Smart Home - System Context

top to bottom direction

!include <C4/C4>
!include <C4/C4_Context>

Person(Customer, "Smart Home Customer", $descr="A smart home customer who has installed sensors and connected to the system.")

System_Ext(HeatingSystem, "Heating system", $descr="Customer-installed sensors and heater.")

System(SmartHomeSystem, "Smart Home System", $descr="Allows customers to control the heating in the home and monitor the temperature.")

Rel(Customer, SmartHomeSystem, "using", $descr="Controls heating and monitors temperature")
Rel(SmartHomeSystem, HeatingSystem, "using", $descr="Receives information from sensors, controls heating")

SHOW_LEGEND(true)
@enduml
```

## Архитектура микросервисов

### Containers 

```puml
@startuml
!theme plain
title Smart Home — Containers
!include <C4/C4>
!include <C4/C4_Context>
!include <C4/C4_Container>

Person(Customer, "Customer")

System_Boundary(SmartHome, "Smart Home") {
	Container(APIGW, "API GateWay")
	Container(Auth, "Auth Service")
	Container(User, "User Service")
	Container(Payment, "Payment Service")
	Container(Notification, "Notification Service")
	Container(Device, "Device Service")
	Container(Telemetry, "Telemetry Service")
	
	ContainerDb(AuthDB, "Auth DB")
	ContainerDb(UserDB, "User DB")
	ContainerDb(PaymentDB, "Payment DB")
	ContainerDb(NotificationDB, "Notification DB")
	ContainerDb(DeviceDB, "Device DB")
	ContainerDb(TelemetryDB, "Telemetry DB")
	
	ContainerQueue(DataBus, "Data Bus", "Kafka")
}

System_Ext(ExtPayment, "Payment System")
System_Ext(UserDevices, "User's Devices")

Rel(Customer, APIGW, "Send requests")
Rel_L(APIGW, Auth, "Verify authentication and authorization")
Rel(Auth, User, "Get authorization data")
Rel(APIGW, Payment, "Manage payment")
Rel(APIGW, Device, "Manage devices")
Rel(APIGW, Telemetry, "View telemetry")
Rel(APIGW, User, "Registration, manage user data")

Rel(Payment, DataBus, "Send payment data")
BiRel(Device, DataBus, "Receive sensor data and send commands for devices")
Rel(Telemetry, DataBus, "Receive sensor data")
Rel(Notification, DataBus, "Receive data")

Rel_U(Notification, Customer, "Notify")

Rel(Auth, AuthDB, "Save tokens and authorization sessions")
Rel(User, UserDB, "Save User Data")
Rel(Notification, NotificationDB, "Save Notifications")
Rel(Payment, PaymentDB, "Save Payment Data")
Rel(Device, DeviceDB, "Save Device Data and Commands")
Rel(Telemetry, TelemetryDB, "Save Telemetry Data")

Rel(Payment, ExtPayment, "Manage payment")
BiRel_U(UserDevices, DataBus, "Send sensor data and receive commands for devices")
@enduml
```

### Components

```puml
@startuml
!theme plain
title Smart Home — Device Service — Components
!include <C4/C4>
!include <C4/C4_Context>
!include <C4/C4_Container>
!include <C4/C4_Component>

Person(Customer, "Customer")
Container(APIGW, "API GateWay")
Container_Boundary(DeviceService, "Device Service") {
	Component(API, "API")
	Component(CommandProcessor, "Command Processor")
	Component(StateManager, "StateManager")
	Component(Rerository, "Rerository")
	Component(Consumer, "Telemetry Consumer")
	Component(Producer, "Telemetry Producer")
}
ContainerDb(DeviceDB, "Device DB")
ContainerQueue(DataBus, "Data Bus", "Kafka")
System_Ext(UserDevices, "User's Devices")

Rel(Customer, APIGW, "Send request")
Rel(APIGW, API, "Manage Device")
Rel(API, CommandProcessor, "")
Rel(API, StateManager, "")
Rel(CommandProcessor, Rerository, "Save command")
Rel(StateManager, Rerository, "Get state")
Rel(Rerository, DeviceDB, "Save and receive data")
Rel(Consumer, CommandProcessor, "Call command")
Rel(CommandProcessor, Producer, "Send command")
Rel(Producer, DataBus, "Produce command")
Rel(DataBus, Consumer, "Receive sensor data")
BiRel(DataBus, UserDevices, "Send sensor data and receive commands for devices")
@enduml
```

### ER

```puml
@startuml
!theme plain
skinparam linetype ortho

package auth as "Auth Setvice" {
	entity "Tokens" as tokens {
		*token_id: number <<generated>>
		*token : text
		--
		*user_id : UUID <<FK>>
	}
}

package user as "User Setvice" {
	entity "Users" as users {
	  *user_id : UUID
	  *name : text
	  *status : enum
	}
}

package device as "Device Setvice" {
	entity "Devices" as devices {
		*device_id : UUID
		*type : enum
		*status : enum
		added_at : timestamp
		--
		*user_id : UUID <<FK>>
	}
	entity "Comands" as commands {
		*command_id : UUID
		*created_at : timestamp
		--
		*device_id : UUID <<FK>>
	}
}

package telemetry as "Telemetry Setvice" {
	entity "Telemetries" as telementies {
		*telemetry_id : UUID
		*created_at : timestamp
		--
		*device_id : UUID <<FK>>
	}
}

package payment as "Payment Setvice" {
	entity "Payments" as payments {
		*payment_id : UUID
		*created_at : timestamp
		updated_at : timestamp
		*status : enum
		--
		*user_id : UUID <<FK>>
	}
}

package notification as "Notification Setvice" {
	entity "Notifications" as notifications {
		*notification_id : UUID
		--
		*user_id : UUID
	}
}

users ||..|{ tokens
users ||..o{ devices
users ||..o{ payments
users }o..o{ notifications
devices ||..o{ commands
devices ||..o{ telementies
@enduml
```



___

# Базовая настройка

## Запуск minikube

[Инструкция по установке](https://minikube.sigs.k8s.io/docs/start/)

```bash
minikube start
```


## Добавление токена авторизации GitHub

[Получение токена](https://github.com/settings/tokens/new)

```bash
kubectl create secret docker-registry ghcr --docker-server=https://ghcr.io --docker-username=<github_username> --docker-password=<github_token> -n default
```


## Установка API GW kusk

[Install Kusk CLI](https://docs.kusk.io/getting-started/install-kusk-cli)

```bash
kusk cluster install
```


## Настройка terraform

[Установите Terraform](https://yandex.cloud/ru/docs/tutorials/infrastructure-management/terraform-quickstart#install-terraform)


Создайте файл ~/.terraformrc

```hcl
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```

## Применяем terraform конфигурацию 

```bash
cd terraform
terraform apply
```

## Настройка API GW

```bash
kusk deploy -i api.yaml
```

## Проверяем работоспособность

```bash
kubectl port-forward svc/kusk-gateway-envoy-fleet -n kusk-system 8080:80
curl localhost:8080/hello
```


## Delete minikube

```bash
minikube delete
```
