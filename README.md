# 📦 MinIO — Роль для развертывания объектного хранилища

## 🔍 Описание

Роль обеспечивает установку и конфигурацию MinIO на виртуальных машинах.

Поддерживаются сценарии:

- **Single Instance** — один узел MinIO.
- **High Availability (HA)** — кластер из нескольких(от 2 до 4) узлов MinIO.
- **Опциональный балансировщик нагрузки** на базе NGINX с единым endpoint’ом.
- Автоматическая подготовка XFS-дисков и каталогов данных.
- Установка MinIO Server и MinIO Client (mc/mcli) из deb-пакетов.
- Генерация systemd-сервисов и env-конфигурации.
- Отдельные действия для создания и удаления бакетов.

---

## ⚙️ Структура проекта

```text
minio/
  ├── defaults/
  │   └── main.yml
  ├── handlers/
  │   └── main.yml
  ├── tasks/
  │   ├── main.yml
  │   ├── install_server.yml
  │   ├── install_client.yml
  │   ├── prepare_disks.yml
  │   ├── prepare_single_disk.yml
  │   ├── configure_server.yml
  │   ├── minio_lb_install.yml
  │   ├── minio_lb_build_backends.yml
  │   ├── minio_lb_configure.yml
  │   ├── build_cluster_nodes.yml
  │   ├── stats.yml
  │   ├── create_minio_buckets.yml
  │   └── delete_minio_bucket.yml
  ├── templates/
  │   ├── minio.env.j2
  │   ├── minio.service.j2
  │   └── minio_lb.nginx.j2
  ├── minio_create_bucket.yml
  ├── minio_delete_bucket.yml
  ├── minio_cluster.yml
  └── README.md
```

---

## 🚀 Возможности роли

### Single Instance

- Развёртывание одного экземпляра MinIO.
- Использование одного или нескольких локальных XFS-дисков.
- Экспорт `connection_url` и `console_url` для доступа к API и консоли.

### High Availability (HA)

- Развёртывание MinIO-кластера на нескольких узлах.
- Формирование общего пула дисков и erasure-code средствами MinIO.
- Автоматическая генерация списка эндпоинтов вида  
  `http://<host>:<port>/<mount>` для всех узлов и дисков.
- Экспорт сведений о топологии:
  - количество узлов и дисков на узел,
  - список `minio_server_cluster_nodes`.

### Балансировщик нагрузки (NGINX)

- Опциональная установка NGINX на отдельном хосте.
- Настройка upstream’ов для:
  - API MinIO (`minio_api`),
  - консоли MinIO (`minio_console`).
- HTTP→API / HTTP→Console редиректы и проброс заголовков.
- Единые точки входа для клиентов:
  - `minio_lb_api_domain` / `minio_lb_console_domain` (если заданы),
  - либо `minio_lb_api_listen` / `minio_lb_console_listen`.

---

## 🔧 Ключевые переменные

Ниже только основные. Полный список — в `defaults/main.yml`.

### Базовые

| Переменная              | Назначение                             | Пример              |
|-------------------------|----------------------------------------|---------------------|
| `minio_install_server`  | Установка MinIO Server                 | `true`              |
| `minio_install_client`  | Установка MinIO Client (mc/mcli)       | `true`              |
| `minio_root_user`       | Администратор MinIO                    | `minio_admin`       |
| `minio_root_password`   | Пароль администратора                  | `SuperSecret123`    |
| `minio_server_port`     | Порт API MinIO                         | `9000`              |
| `minio_console_port`    | Порт веб-консоли                       | `9001`              |
| `minio_alias`           | Имя alias для mc                       | `myminio`           |
| `minio_server_deb_url`  | URL deb-пакета сервера                 | `<URL>`             |
| `minio_client_deb_url`  | URL deb-пакета клиента                 | `<URL>`             |

### Диски и данные

Ожидается использование XFS и `noatime`.

| Переменная             | Назначение                                         |
|------------------------|----------------------------------------------------|
| `minio_server_datadirs`| Список каталогов данных MinIO (если заданы явно). |
| `minio_extra_disks`    | Описание дополнительных дисков для форматирования и монтирования (используется в `prepare_disks.yml`). |

Если `minio_extra_disks` не задан, роль может подготовить дефолтный каталог (например, `/app/d1`) в зависимости от логики `prepare_disks.yml`.

### Режим работы кластера

| Переменная                  | Назначение                                       | Значения                              |
|-----------------------------|--------------------------------------------------|---------------------------------------|
| `minio_cluster_mode`        | Режим развертывания                              | `"Single Instance"` \| `"High Availability"` |
| `minio_cluster_nodes_group` | Имя инвентарной группы для MinIO-узлов          | `node` (по умолчанию)                |

Группа `minio_cluster_nodes_group` используется для:

- определения списка узлов кластера;
- построения backend’ов для NGINX;
- вычисления топологии в `stats.yml` и `build_cluster_nodes.yml`.

### Балансировщик нагрузки

Активируется явно.

| Переменная                | Назначение                                      | Пример                     |
|---------------------------|-------------------------------------------------|----------------------------|
| `minio_enable_lb`         | Включить установку и конфиг NGINX LB           | `true`/`false`             |
| `minio_lb_group`          | Группа хостов, где поднимается NGINX           | `lb`                       |
| `minio_lb_backends`       | Явный список backend-хостов (IP/имена)         | `["10.0.0.1","10.0.0.2"]`  |
| `minio_lb_api_listen`     | Адрес/порт для API на LB                       | `0.0.0.0:9000`             |
| `minio_lb_console_listen` | Адрес/порт для Console на LB                   | `0.0.0.0:9001`             |
| `minio_lb_api_port`       | Порт backend’ов API MinIO                      | `9000`                     |
| `minio_lb_console_port`   | Порт backend’ов Console                        | `9001`                     |
| `minio_lb_api_domain`     | DNS-имя для API (опционально)                  | `minio.example.com`        |
| `minio_lb_console_domain` | DNS-имя для Console (опционально)              | `minio-console.example.com`|
| `minio_lb_nginx_conf`     | Путь для сгенерированного конфига NGINX        | `/etc/nginx/conf.d/minio.conf` |

Backend’ы:

- Если `minio_lb_backends` не задан, используются хосты из `minio_cluster_nodes_group`.

---

---

## 🧾 Действия с бакетами

Отдельные плейбуки/таски:

- `tasks/create_minio_buckets.yml` — идемпотентное создание бакетов.
- `tasks/delete_minio_bucket.yml` — безопасное удаление бакетов.
- `minio_create_bucket.yml` / `minio_delete_bucket.yml` — примеры обёрток для вызова этих тасок.

Используют `mc` и настроенный `minio_alias`. Пароли в командах не светятся — авторизация идёт через alias.

---

## ✅ Проверки после развертывания

### 1. Сервис MinIO на всех узлах

На каждом MinIO-узле:

systemctl is-active minio
systemctl is-enabled minio

Ожидание:
- is-active → active
- is-enabled → enabled

### 2. Диски и каталоги данных

Проверить, что каталоги данных (например, /app/d1, /app/d2, ...) существуют и примонтированы корректно:

mount | grep /app/
ls -ld /app/*

Ожидания по каждому каталогу данных:
- файловая система xfs;
- опция noatime;
- владелец и группа соответствуют пользователю MinIO (по умолчанию minio:minio).

### 3. Health-эндпоинты MinIO

Проверка напрямую на каждой ноде:

curl -L -I http://<node_ip>:9000/minio/health/live
curl -L -I http://<node_ip>:9000/minio/health/ready

Проверка через балансировщик (если используется):

curl -L -I http://<API_endpoint>/minio/health/live
curl -L -I http://<API_endpoint>/minio/health/ready

Ожидаемый код для /live и /ready: 200 OK.

(Опционально для HA: curl -L -I http://<API-endpoint>/minio/health/cluster — 200 OK.)

### 4. Проверка NGINX (если включён LB)

На хосте балансировщика:

systemctl is-active nginx
test -f /etc/nginx/conf.d/minio.conf
nginx -t

Ожидание:
- nginx активен;
- конфиг minio.conf существует;
- nginx -t завершается без ошибок;
- health-запросы через <API_endpoint> возвращают 200 OK (см. пункт 3).

### 5. Проверка alias myminio

sudo mc alias ls myminio

Ожидание:
- alias myminio существует;
- endpoint в alias соответствует адресу узла.

### 6. Топология кластера (mc admin info)

Для HA-кластера:

sudo mc admin info myminio

Проверяем:
- количество Servers соответствует ожидаемому числу узлов;
- state у всех серверов online;
- количество дисков на сервер совпадает с конфигурацией;
- нет offline/errored дисков.

Подтверждает корректную кластерную конфигурацию.

### 7. Базовый доступ к S3 API

Проверка без создания бакетов:

sudo mc ls myminio

Ожидание:
- команда выполняется без ошибок;
- выводится список существующих бакетов (если уже созданы отдельными действиями) или пустой список.

При наличии уже созданного бакета:

sudo mc ls myminio/<bucket-name>
sudo mc stat myminio/<bucket-name>

Ожидание:
- успешное выполнение без AccessDenied и сетевых ошибок.

### 8. Доступ к консоли MinIO
1. Открыть URL в браузере.
2. Войти под minio_root_user / minio_root_password.
3. Проверить:
   - все бакеты созданные отображаются;
---