# Домашнее задание к занятию 4 «Работа с roles» — Мешков Сергей

---

## 1. Описание проекта

В рамках выполнения задания монолитный Playbook из предыдущих занятий был разбит на **Ansible-роли**. Это позволило сделать код переиспользуемым, структурированным и удобным для поддержки.

Проект развертывает три взаимосвязанных сервиса:
1.  **ClickHouse** — колоночная база данных для хранения логов.
2.  **Vector** — агент сбора логов, который генерирует тестовые данные и отправляет их в ClickHouse.
3.  **LightHouse** — веб-интерфейс для просмотра данных из ClickHouse.

### Ссылки на репозитории

*   **Роль ClickHouse:** [https://github.com/meshkov-sergey/clickhouse-role](https://github.com/meshkov-sergey/clickhouse-role)
*   **Роль Vector:** [https://github.com/meshkov-sergey/vector-role](https://github.com/meshkov-sergey/vector-role)
*   **Роль LightHouse:** [https://github.com/meshkov-sergey/lighthouse-role](https://github.com/meshkov-sergey/lighthouse-role)
*   **Репозиторий с Playbook:** [https://github.com/meshkov-sergey/netology-ans-04-hw](https://github.com/meshkov-sergey/netology-ans-04-hw)

---

## 2. Архитектура и структура проекта

```text
netology-ans-04-hw/
├── site.yml                 # Главный Playbook (вызов ролей)
├── inventory/
│   └── prod.yml             # Инвентарь (Docker-контейнеры)
├── requirements.yml         # Список ролей для скачивания с GitHub
├── .gitignore               # Игнорируем папку roles/ и мусор
└── roles/                   # Скачанные роли (не в Git, ставятся через requirements.yml)
    ├── clickhouse-role/
    ├── vector-role/
    └── lighthouse-role/
```

**Ключевая идея:** Папка `roles/` не хранится в репозитории Playbook (она в `.gitignore`). Вместо этого она скачивается с GitHub через `requirements.yml` командой:
```bash
ansible-galaxy install -r requirements.yml -p roles/
```

---

## 3. Разбор ролей

### 3.1. `clickhouse-role`
*   **Задачи:**
    *   Скачивает RPM-пакеты (common-static, server, client) версии 22.10.1.1877.
    *   Устанавливает их через `dnf` (с флагом `disable_gpg_check: true`, так как локальные файлы не подписаны).
    *   Создает `network.xml` с `listen_host: 0.0.0.0` (чтобы ClickHouse слушал внешние интерфейсы).
    *   Создает базу `logs` и таблицу `logs.logs` (с колонками `timestamp String` и `message String`).
*   **Переменные (defaults):** `clickhouse_version`, `clickhouse_packages`.
*   **Хэндлер:** `Start clickhouse service` (перезапускает службу).

### 3.2. `vector-role`
*   **Задачи:**
    *   Скачивает архив Vector (`tar.gz`) с официального сайта.
    *   Распаковывает в `/opt/vector`, создает симлинк `/usr/local/bin/vector`.
    *   Разворачивает конфиг из шаблона `vector.j2` (подставляет адрес ClickHouse и настройки).
    *   Запускает Vector через `command` с `async` (так как в контейнере нет `systemd`).
*   **Переменные (defaults):** `vector_version`, `vector_install_dir`, `vector_config_dir`, `vector_data_dir`, `clickhouse_host` (вычисляется из инвентаря).
*   **Хэндлер:** `Restart vector service` (перезапускает процесс).

### 3.3. `lighthouse-role`
*   **Задачи:**
    *   Устанавливает Nginx (через `apt` для Debian/Ubuntu).
    *   Скачивает статику LightHouse с GitHub, распаковывает в `/var/www/lighthouse`.
    *   Разворачивает конфиг Nginx из шаблона `lighthouse.j2`.
    *   Включает сайт (симлинк в `sites-enabled`), удаляет дефолтный сайт.
    *   Запускает Nginx через `command` (так как нет `systemd`).
*   **Переменные (defaults):** `lighthouse_dir`, `nginx_conf_dir`.
*   **Хэндлер:** `Reload nginx` (перезагружает конфиг).

---

## 4. Связка трех сервисов (как это работает)

1.  **Vector** генерирует тестовые логи (JSON с полями `host`, `user_id`, `timestamp` и т.д.) и отправляет их по HTTP в **ClickHouse** (на порт 8123).
2.  **ClickHouse** принимает эти логи и сохраняет их в таблицу `logs.logs`.
3.  **LightHouse** (веб-интерфейс) подключается к ClickHouse по API (порт 8123) и отображает содержимое таблицы в браузере.

**В реальной жизни:** Vector ставится на каждый сервер, собирает реальные логи (access.log, error.log) и шлет в централизованный ClickHouse. А инженеры через LightHouse (или Grafana) смотрят эти логи в одном окне.

---

## 5. Инфраструктура (Docker)

Для тестирования используются три Docker-контейнера (из предыдущего задания):
*   `clickhouse-01` — CentOS 8 с systemd (порты 8123, 9000).
*   `vector-01` — Ubuntu с Python.
*   `lighthouse-01` — Ubuntu с Nginx и Python (порт 8080).

Запуск:
```bash
docker compose up -d --build
```

---

## 6. Проверка и идемпотентность

### Запуск с `--check` (Dry-run)
```bash
ansible-playbook -i inventory/prod.yml site.yml --check
```
*Флаг `--check` проверяет синтаксис и показывает, какие изменения были бы внесены.*

### Запуск с `--diff` (Реальный прогон)
```bash
ansible-playbook -i inventory/prod.yml site.yml --diff
```
*Флаг `--diff` показывает разницу в файлах при выполнении задач.*

### Повторный запуск (Идемпотентность)
Повторный запуск показал `changed=0` для всех хостов (`clickhouse-01`, `vector-01`, `lighthouse-01`). Playbook идемпотентен.

---

## 7. Флаги Ansible, использованные в работе

*   `--check` (`-C`) — режим «имитации».
*   `--diff` (`-D`) — показывает разницу в файлах.
*   `-i` (`--inventory`) — указать путь к инвентарю.

---
## Автор
- **Студент**: Мешков Сергей
- **Курс**: Netology, DevOps
