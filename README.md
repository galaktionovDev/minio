# 📦 MinIO — Объектное хранилище

## 🔍 Описание

Роль MinIO обеспечивает установку и настройку объектного хранилища MinIO
Решение поддерживает:

- автоматическую установку MinIO Server и MinIO Client (mcli);
- подготовку и монтирование дополнительных дисков XFS;
- генерацию всех необходимых конфигураций и сервисов systemd;
- экспорт параметров подключения для UI;
- отдельные действия для создания и удаления бакетов.

---

## ⚙️ Структура проекта

```
      ├── defaults/
      │   └── main.yml             
      ├── vars/
      │   └── main.yml             
      ├── tasks/
      │   ├── main.yml             
      │   ├── install_server.yml    
      │   ├── install_client.yml   
      │   ├── prepare_disks.yml     
      │   ├── prepare_single_disk.yml
      │   ├── configure_server.yml   
      │   ├── create_minio_buckets.yml 
      │   └── delete_minio_bucket.yml  
      ├── handlers/
      │   └── main.yml              
      ├── templates/
      │   ├── minio.env.j2          
      │   ├── minio.service.j2      
      │   └── minio_policy.json.j2
      ├── minio_create_bucket.yml 
      ├── minio_delete_bucket.yml 
      ├── minio_standalone.yml   
      └── README.md                  
```

---

## 🧩 Основные параметры

| Переменная              | Назначение                       | Пример       |
|-------------------------|---------------------------------|-------------|
| minio_install_server     | Устанавливать сервер MinIO       | true        |
| minio_install_client     | Устанавливать клиент MinIO       | true        |
| minio_root_user          | Имя администратора               | admin       |
| minio_root_password      | Пароль администратора            | secret123   |
| minio_server_port        | Порт API MinIO                   | 9000        |
| minio_console_port       | Порт веб-консоли                 | 9001        |
| minio_server_deb_url     | URL deb-пакета сервера           | `<Nexus URL>` |
| minio_client_deb_url     | URL deb-пакета клиента           | `<Nexus URL>` |
| minio_extra_disks        | Список дисков для форматирования и монтирования | см. ниже |

---
## 🧰 Проверка функциональности

| № | Проверка | Команды | ✅ Ожидаемый результат |
|--:|-----------|----------|------------------------|
| 1 | **Проверка состояния службы** | `sudo systemctl status minio`<br>`sudo systemctl is-enabled minio` | Статус `active (running)` и `enabled`. |
| 2 | **Проверка бинарников и путей** | `which minio`<br>`which mc`<br>  `which mcli`<br>`ls -l /usr/local/bin/minio* && ls -l /usr/local/bin/mc*` | Бинарники существуют и имеют права на исполнение. |
| 3 | **Проверка структуры каталогов** | `sudo ls -l /app/disk*`<br>`ls -ld /etc/minio/ && ls -ld /etc/systemd/system/minio.service` | Все диски `/app/disk*` примонтированы, `/etc/minio` существует, конфигурационные файлы корректны. |
| 4 | **Проверка конфигурации среды MinIO** | `sudo cat /etc/minio/minio.conf` | Содержит переменные:<br> `MINIO_VOLUMES="/app/disk*"`<br>`MINIO_ROOT_USER=<user>`<br>`MINIO_ROOT_PASSWORD=<password>`|
| 5 | **Проверка доступности API** | `curl -I http://<IP-addr>>:9000/minio/health/live` | HTTP-код ответа `200`. |


---

## 🧾 Действия

| Действие       | Плейбук                        | Назначение                      |
|----------------|--------------------------------|--------------------------------|
| Создать бакет   | `tasks/create_minio_buckets.yml` | Идемпотентное создание бакета |
| Удалить бакет   | `tasks/delete_minio_bucket.yml`  | Безопасное удаление бакета    |

---

## 🧱 Экспорт переменных

Роль автоматически передает данные для UI:

```yaml
set_stats:
  data:
    connection_url: "http://<host>:9000"
    console_url:    "http://<host>:9001"
    access_key:     "<minio_root_user>"
```

Пароли и секреты в вывод не попадают.

---