# Moltbot Installation Guide

Полная инструкция по установке Moltbot на GCP VM с Telegram интеграцией.

> **Disclaimer:** Данная инструкция предоставляется "как есть" без каких-либо гарантий. Moltbot — это сторонний проект с известными проблемами безопасности. Используйте на свой страх и риск. Автор инструкции не несёт ответственности за возможные потери данных, утечки credentials или другие последствия.

---

## Prerequisites

Перед началом убедитесь, что у вас есть:

- **GCP аккаунт** с активным billing (или free trial credits)
- **Tailscale аккаунт** (бесплатный на tailscale.com)
- **Telegram аккаунт** для создания бота
- **gcloud CLI** установлен локально
- **Базовые знания Linux** (терминал, SSH, редактирование файлов)
- **ChatGPT Plus / OpenAI API** или другой AI провайдер

### Альтернативы GCP

Эта инструкция написана для GCP, но Moltbot можно запустить на любом Linux-сервере.

Основные требования: 2+ vCPU, 4GB+ RAM, Ubuntu 22.04/24.04.

---

## Оглавление

1. [GCP Setup](#1-gcp-setup)
2. [VM Creation](#2-vm-creation)
3. [Tailscale Setup](#3-tailscale-setup)
4. [Moltbot Installation](#4-moltbot-installation)
5. [Telegram Configuration](#5-telegram-configuration)
6. [Management Commands](#6-management-commands)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. GCP Setup

### Создание проекта

```bash
# Создать проект (без организации)
gcloud projects create <PROJECT_ID> --name="<PROJECT_NAME>"

# ИЛИ с организацией (если есть)
gcloud projects create <PROJECT_ID> --name="<PROJECT_NAME>" --organization=<ORGANIZATION_ID>

# Привязать billing
gcloud billing projects link <PROJECT_ID> --billing-account=<BILLING_ACCOUNT_ID>

# Включить Billing Budgets API
gcloud services enable billingbudgets.googleapis.com --project=<PROJECT_ID>

# Создать бюджет с алертами (опционально, но рекомендуется)
gcloud billing budgets create \
  --billing-account=<BILLING_ACCOUNT_ID> \
  --display-name="<PROJECT_ID>-budget" \
  --budget-amount=100USD \
  --filter-projects="projects/<PROJECT_ID>" \
  --threshold-rule=percent=50 \
  --threshold-rule=percent=90 \
  --threshold-rule=percent=100
```

### Включение API

```bash
gcloud services enable compute.googleapis.com --project=<PROJECT_ID>

# Опционально: для Gemini API
gcloud services enable generativelanguage.googleapis.com --project=<PROJECT_ID>
```

### Создание Gemini API Key (опционально)

```bash
gcloud services api-keys create --display-name="Gemini API Key" --project=<PROJECT_ID>
```

---

## 2. VM Creation

### Параметры VM

| Параметр | Значение | Примечание |
|----------|----------|------------|
| Имя | moltbot | Можно любое |
| Тип | e2-standard-2 (2 vCPU, 8GB RAM) | Минимум для комфортной работы |
| Диск | 100GB SSD | Можно меньше (20-50GB) |
| OS | Ubuntu 24.04 LTS | Или 22.04 LTS |
| Регион | us-central1-a | Ближе к AI API серверам |
| Публичный IP | Временный | Удаляется после настройки Tailscale |

### Создание VM

```bash
gcloud compute instances create moltbot \
  --project=<PROJECT_ID> \
  --zone=us-central1-a \
  --machine-type=e2-standard-2 \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=100GB \
  --boot-disk-type=pd-ssd \
  --tags=allow-ssh \
  --metadata=enable-oslogin=TRUE
```

### Firewall для SSH

```bash
gcloud compute firewall-rules create allow-ssh \
  --project=<PROJECT_ID> \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=allow-ssh
```

---

## 3. Tailscale Setup

### Зачем Tailscale?

Tailscale создаёт защищённую сеть между вашими устройствами. Это позволяет:
- Убрать публичный IP с сервера (безопаснее)
- Подключаться к серверу из любой точки мира
- Не открывать порты в firewall

### Установка Tailscale на VM

```bash
# SSH на VM (с временным публичным IP)
gcloud compute ssh moltbot --zone=us-central1-a --project=<PROJECT_ID>

# Установить Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Запустить и получить ссылку авторизации
sudo tailscale up
# Откройте ссылку в браузере и авторизуйте устройство

# Включить Tailscale SSH (важно!)
sudo tailscale up --ssh
```

### Установка Tailscale локально

Установите Tailscale на вашем компьютере (tailscale.com/download) и авторизуйтесь в том же аккаунте.

### Cloud NAT (для outbound без публичного IP)

После настройки Tailscale нужен Cloud NAT для исходящего трафика:

```bash
# Создать Cloud Router
gcloud compute routers create nat-router \
  --project=<PROJECT_ID> \
  --network=default \
  --region=us-central1

# Создать Cloud NAT
gcloud compute routers nats create nat-config \
  --project=<PROJECT_ID> \
  --router=nat-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

### Удаление публичного IP

```bash
gcloud compute instances delete-access-config moltbot \
  --zone=us-central1-a \
  --project=<PROJECT_ID> \
  --access-config-name="external-nat"
```

### Проверка подключения

```bash
# С локальной машины (Tailscale должен быть запущен)
tailscale status          # Должен показать moltbot
tailscale ping moltbot    # Должен отвечать
ssh root@moltbot          # Подключение через Tailscale
```

---

## 4. Moltbot Installation

### Создание пользователя

```bash
ssh root@moltbot
useradd -m -s /bin/bash moltbot
```

### Установка Node.js 24

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
apt-get install -y nodejs
node --version  # Должно быть v24.x.x
```

### Установка pnpm

```bash
npm install -g pnpm
pnpm --version  # Должно быть 10.x.x
```

### Установка Moltbot

> **⚠️ ВАЖНО:** Используйте только официальный инсталлятор!
>
> NPM пакет `moltbot` — это placeholder/squatter, **НЕ официальный пакет**.
> Установка через `npm install -g moltbot` установит пустой/вредоносный пакет.

```bash
su - moltbot
curl -fsSL https://molt.bot/install-cli.sh | bash
```

### Настройка PATH

```bash
# Добавить в ~/.bashrc
echo 'export PATH="$HOME/.clawdbot/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Проверить установку
clawdbot --version
```

### Onboarding

```bash
clawdbot onboard
```

Во время onboarding выберите:
1. **Model**: OpenAI → OpenAI Codex (ChatGPT OAuth) — или другой провайдер
2. **Channel**: Telegram
3. **Skills**: выберите нужные (можно пропустить)
4. **Hooks**: boot-md, command-logger, session-memory (рекомендуется)

### Права доступа (security)

```bash
chmod 700 ~/.clawdbot
```

---

## 5. Telegram Configuration

### Создание бота

1. Откройте @BotFather в Telegram
2. Отправьте `/newbot`
3. Введите имя бота (например: "My Assistant")
4. Введите username (например: "my_assistant_bot")
5. Скопируйте полученный токен

### Добавление токена в Moltbot

Токен вводится при onboarding. Если пропустили — добавьте вручную в `~/.clawdbot/clawdbot.json`.

### Pairing (первое подключение)

При первом сообщении боту:
1. Бот выдаст pairing code (например: `ABC123`)
2. На сервере выполните:

```bash
clawdbot pairing approve telegram <CODE>
```

После этого бот будет отвечать на ваши сообщения.

---

## 6. Management Commands

### Gateway (основной процесс)

```bash
# Запустить Gateway (в фоне)
nohup clawdbot gateway run > ~/.clawdbot/gateway.log 2>&1 &

# Остановить
clawdbot gateway stop

# Проверить статус
clawdbot channels status
```

### Каналы

```bash
# Статус всех каналов
clawdbot channels status

# Логи каналов
clawdbot channels logs | tail -50
```

### Security

```bash
# Аудит безопасности
clawdbot security audit
clawdbot security audit --deep

# Автоисправление проблем
clawdbot security audit --fix
```

### Pairing

```bash
# Список ожидающих подтверждения
clawdbot pairing list

# Одобрить пользователя
clawdbot pairing approve telegram <CODE>
```

### Диагностика

```bash
clawdbot doctor
```

---

## 7. Troubleshooting

### Gateway не запускается

```bash
# Проверить, не запущен ли уже
ps aux | grep clawdbot

# Убить процесс
pkill -u moltbot -f clawdbot-gateway

# Запустить заново
nohup clawdbot gateway run > ~/.clawdbot/gateway.log 2>&1 &
```

### Telegram не отвечает

```bash
# Проверить статус
clawdbot channels status

# Посмотреть логи
clawdbot channels logs | tail -50

# Перезапустить Gateway
pkill -u moltbot -f clawdbot-gateway
nohup clawdbot gateway run > ~/.clawdbot/gateway.log 2>&1 &
```

### Tailscale offline

```bash
# На VM
sudo systemctl restart tailscaled
sudo tailscale up --ssh

# Проверить
tailscale status
```

### "fetch failed" ошибки

Обычно проблема с сетью. Проверьте:

```bash
# Интернет работает?
curl -I https://api.openai.com

# Cloud NAT работает?
curl -I https://google.com
```

### VM не имеет интернета после удаления публичного IP

Убедитесь, что Cloud NAT настроен (см. раздел 3).

---

## Quick Reference

### SSH доступ

```bash
ssh root@moltbot           # Через Tailscale
su - moltbot               # Переключиться на пользователя
export PATH="$HOME/.clawdbot/bin:$PATH"
```

### Control UI через туннель

```bash
# На локальной машине
ssh -N -L 18789:127.0.0.1:18789 root@moltbot

# Открыть в браузере
http://localhost:18789/?token=<TOKEN>
```

### Важные пути

| Путь | Описание |
|------|----------|
| `~/.clawdbot/` | Конфигурация и состояние |
| `~/.clawdbot/clawdbot.json` | Основной конфиг |
| `~/.clawdbot/bin/clawdbot` | CLI бинарник |
| `~/clawd/` | Workspace агента |
| `/tmp/clawdbot/` | Логи |

---

## Security Notes

### Что делает эта инструкция для безопасности

1. **Нет публичного IP** — сервер недоступен из интернета
2. **Tailscale** — шифрованное соединение между устройствами
3. **Cloud NAT** — только исходящий трафик, входящий заблокирован
4. **Dedicated user** — `moltbot` с ограниченными правами
5. **Права 700** — конфиги недоступны другим пользователям
6. **Pairing mode** — новые пользователи требуют одобрения
7. **Gateway на loopback** — Control UI недоступен извне

### ⚠️ Известные риски Moltbot

> **Внимание:** Moltbot имеет архитектурные проблемы безопасности:
>
> - **Credentials в plaintext** — API ключи хранятся в обычных файлах
> - **Полный доступ к системе** — AI агент может выполнять любые команды
> - **Supply chain атаки** — вредоносные skills могут быть опубликованы
> - **Нет sandboxing по умолчанию** — агент не изолирован
>
> **Рекомендации:**
> - Используйте изолированную VM (не ставьте на основной сервер)
> - Не храните критичные данные на этой VM
> - Регулярно запускайте `clawdbot security audit`
> - Ротируйте API ключи при подозрении на утечку
> - Не устанавливайте непроверенные skills

---

## Стоимость (GCP)

| Компонент | Стоимость/мес |
|-----------|---------------|
| e2-standard-2 | ~$49 |
| 100GB SSD | ~$17 |
| Cloud NAT | ~$1 |
| **Итого** | **~$67** |

Можно сэкономить:
- e2-small (2 vCPU, 2GB RAM) — ~$12/мес
- 30GB SSD — ~$5/мес
- **Минимум: ~$18/мес**

Рекомендуется настроить budget alert.

---

## Placeholders

Замените в командах:

| Placeholder | Описание | Где взять |
|-------------|----------|-----------|
| `<PROJECT_ID>` | ID проекта GCP | Придумайте уникальный (латиница, цифры, дефисы) |
| `<PROJECT_NAME>` | Название проекта | Любое понятное вам |
| `<ORGANIZATION_ID>` | ID организации | `gcloud organizations list` (опционально) |
| `<BILLING_ACCOUNT_ID>` | ID billing account | `gcloud billing accounts list` |
| `<TAILSCALE_IP>` | IP в Tailscale | `tailscale status` |
| `<TOKEN>` | Gateway token | Выводится при onboarding |
| `<CODE>` | Pairing code | Выдаёт бот при первом сообщении |
