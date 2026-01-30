# OpenClaw Installation Guide

Полная инструкция по установке OpenClaw на GCP VM с Telegram интеграцией.

> **Disclaimer:** Данная инструкция предоставляется "как есть" без каких-либо гарантий. Используйте на свой страх и риск. Автор инструкции не несёт ответственности за возможные потери данных, утечки credentials или другие последствия.

---

## Что такое OpenClaw?

OpenClaw — это messaging gateway, который соединяет AI-агентов с популярными чат-платформами: WhatsApp, Telegram, Discord, iMessage и Mattermost. Позволяет взаимодействовать с AI-агентами через привычные мессенджеры.

**Ключевые возможности:**
- Мульти-платформенная поддержка (WhatsApp, Telegram, Discord, iMessage)
- Групповые чаты с mention-активацией
- Поддержка медиа (изображения, аудио, документы)
- WebChat и macOS/мобильные приложения
- MIT лицензия — полностью open source

---

## Важные замечания по безопасности

**Прочитайте перед установкой!**

### Архитектурные особенности

- **Credentials в файлах** — API ключи хранятся в файловой системе
- **Доступ к системе** — AI агент может выполнять команды в рамках своих прав
- **Prompt injection** — модели уязвимы к манипуляциям через входящие сообщения

### Рекомендации по безопасности

1. **Gateway только на loopback** — никогда не открывайте порт 18789 в интернет
2. **Используйте изолированную VM** — не ставьте на основной рабочий компьютер
3. **Pairing mode по умолчанию** — новые пользователи требуют одобрения
4. **Регулярно запускайте** `openclaw security audit`
5. **Права доступа 600/700** на конфиги и credentials
6. **Используйте современные модели** (Opus 4.5) для агентов с инструментами

### Полезные ресурсы

- [Официальная документация](https://docs.openclaw.ai/)
- [Security Best Practices](https://docs.openclaw.ai/gateway/security)

---

## Prerequisites

Перед началом убедитесь, что у вас есть:

- **GCP аккаунт** с активным billing (или free trial credits)
- **Tailscale аккаунт** (бесплатный на tailscale.com)
- **Telegram аккаунт** для создания бота
- **gcloud CLI** установлен локально
- **Базовые знания Linux** (терминал, SSH, редактирование файлов)
- **API ключ AI провайдера** (Anthropic, OpenAI, или другой)

### Системные требования

- Node.js 22 или выше
- 2+ vCPU, 4GB+ RAM
- Ubuntu 22.04/24.04

### Альтернативы GCP

Эта инструкция написана для GCP, но OpenClaw можно запустить на любом Linux-сервере.

---

## Оглавление

1. [GCP Setup](#1-gcp-setup)
2. [VM Creation](#2-vm-creation)
3. [Tailscale Setup](#3-tailscale-setup)
4. [OpenClaw Installation](#4-openclaw-installation)
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
```

---

## 2. VM Creation

### Параметры VM

| Параметр | Значение | Примечание |
|----------|----------|------------|
| Имя | openclaw | Можно любое |
| Тип | e2-standard-2 (2 vCPU, 8GB RAM) | Минимум для комфортной работы |
| Диск | 100GB SSD | Можно меньше (20-50GB) |
| OS | Ubuntu 24.04 LTS | Или 22.04 LTS |
| Регион | us-central1-a | Ближе к AI API серверам |
| Публичный IP | Временный | Удаляется после настройки Tailscale |

### Создание VM

```bash
gcloud compute instances create openclaw \
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
gcloud compute ssh openclaw --zone=us-central1-a --project=<PROJECT_ID>

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
gcloud compute instances delete-access-config openclaw \
  --zone=us-central1-a \
  --project=<PROJECT_ID> \
  --access-config-name="external-nat"
```

### Проверка подключения

```bash
# С локальной машины (Tailscale должен быть запущен)
tailscale status          # Должен показать openclaw
tailscale ping openclaw   # Должен отвечать
ssh root@openclaw         # Подключение через Tailscale
```

---

## 4. OpenClaw Installation

### Создание пользователя

```bash
ssh root@openclaw
useradd -m -s /bin/bash openclaw
```

### Установка Node.js 22

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs
node --version  # Должно быть v22.x.x
```

### Установка OpenClaw

**Рекомендуемый способ (официальный инсталлятор):**

```bash
su - openclaw
curl -fsSL https://openclaw.bot/install.sh | bash
```

**Альтернатива через npm:**

```bash
npm install -g openclaw@latest
```

### Настройка PATH

```bash
# Добавить в ~/.bashrc
echo 'export PATH="$HOME/.openclaw/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Проверить установку
openclaw --version
```

### Onboarding

```bash
openclaw onboard --install-daemon
```

Wizard настроит:
1. **Аутентификацию** — API ключ Anthropic (рекомендуется) или другой провайдер
2. **Gateway** — фоновый сервис
3. **Каналы** — Telegram, WhatsApp, Discord и др.
4. **Security defaults** — базовые настройки безопасности

### Права доступа (security)

```bash
chmod 700 ~/.openclaw
```

---

## 5. Telegram Configuration

### Создание бота

1. Откройте @BotFather в Telegram
2. Отправьте `/newbot`
3. Введите имя бота (например: "My Assistant")
4. Введите username (должен заканчиваться на "bot")
5. Скопируйте полученный токен

**Опциональные настройки в BotFather:**
- `/setjoingroups` — разрешить/запретить добавление в группы
- `/setprivacy` — управление видимостью сообщений в группах

### Добавление токена

**Через переменную окружения:**
```bash
export TELEGRAM_BOT_TOKEN="your_token_here"
```

**Или в конфиге `~/.openclaw/openclaw.json`:**
```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your_token_here"
    }
  }
}
```

### Pairing (первое подключение)

При первом сообщении боту:
1. Бот выдаст pairing code (действует 1 час)
2. На сервере выполните:

```bash
openclaw pairing approve telegram <CODE>
```

**Альтернатива — allowlist:**
```json5
{
  channels: {
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: [YOUR_TELEGRAM_USER_ID]
    }
  }
}
```

Узнать свой user ID: отправьте сообщение боту и посмотрите в логах `openclaw logs --follow`.

### Настройка групп

```json5
{
  channels: {
    telegram: {
      groups: { "*": { requireMention: true } }
    }
  }
}
```

Отключите Privacy Mode в BotFather (`/setprivacy`) чтобы бот видел все сообщения в группе.

---

## 6. Management Commands

### Gateway (основной процесс)

```bash
# Статус Gateway
openclaw gateway status

# Запустить Gateway (если не daemon)
openclaw gateway --port 18789 --verbose

# Принудительно перезапустить (если порт занят)
openclaw gateway --force

# Управление daemon (systemd)
openclaw gateway install    # Установить как systemd сервис
openclaw gateway stop       # Остановить
openclaw gateway restart    # Перезапустить

# Открыть Dashboard в браузере
openclaw dashboard
# Или вручную: http://127.0.0.1:18789/
```

### Каналы

```bash
# Статус всех каналов
openclaw channels status

# WhatsApp QR-код для авторизации
openclaw channels login

# Логи (live)
openclaw logs --follow
```

### Security

```bash
# Базовый аудит
openclaw security audit

# Глубокий аудит (проверяет Gateway)
openclaw security audit --deep

# Автоисправление проблем
openclaw security audit --fix
```

### Pairing

```bash
# Список ожидающих подтверждения
openclaw pairing list telegram

# Одобрить пользователя
openclaw pairing approve telegram <CODE>
```

### Диагностика

```bash
openclaw doctor
openclaw status
openclaw health    # Альтернатива status
```

### Cron и память

```bash
openclaw cron list           # Список cron jobs
openclaw cron run <id>       # Запустить вручную
openclaw memory search "query"  # Поиск по памяти
```

---

## 7. Troubleshooting

### Gateway не запускается

```bash
# Проверить, не запущен ли уже
ps aux | grep openclaw

# Вариант 1: Принудительно перезапустить (закроет старый процесс)
openclaw gateway --force --verbose

# Вариант 2: Убить вручную и запустить
pkill -f openclaw
openclaw gateway --port 18789 --verbose
```

### Telegram не отвечает

```bash
# Проверить статус
openclaw channels status

# Посмотреть логи
openclaw logs --follow

# Проверить токен
echo $TELEGRAM_BOT_TOKEN
```

### Tailscale offline

```bash
# На VM
sudo systemctl restart tailscaled
sudo tailscale up --ssh

# Проверить
tailscale status
```

### Проблемы с сетью

```bash
# Интернет работает?
curl -I https://api.anthropic.com

# Cloud NAT работает?
curl -I https://google.com
```

### VM не имеет интернета после удаления публичного IP

Убедитесь, что Cloud NAT настроен (см. раздел 3).

---

## Quick Reference

### SSH доступ

```bash
ssh root@openclaw          # Через Tailscale
su - openclaw              # Переключиться на пользователя
export PATH="$HOME/.openclaw/bin:$PATH"
```

### Dashboard через туннель

**Вариант 1: SSH туннель**
```bash
# На локальной машине
ssh -N -L 18789:127.0.0.1:18789 root@openclaw

# Открыть в браузере
http://localhost:18789/
```

**Вариант 2: Tailscale Serve (рекомендуется)**

Добавить в `~/.openclaw/openclaw.json`:
```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" }
  }
}
```

Доступ: `https://openclaw.<tailnet-name>.ts.net/`

### Важные пути

| Путь | Описание |
|------|----------|
| `~/.openclaw/` | Конфигурация и состояние |
| `~/.openclaw/openclaw.json` | Основной конфиг (permissions 600) |
| `~/.openclaw/credentials/` | OAuth и credentials |
| `~/.openclaw/agents/<agentId>/` | Данные агентов |
| `~/.openclaw/workspace/` | Workspace агента (SOUL.md, MEMORY.md и др.) |
| `~/.openclaw/workspace/skills/` | Per-agent skills |
| `~/.openclaw/skills/` | Shared skills (все агенты) |
| `/tmp/openclaw/` | Логи

---

## Security Checklist

### Что делает эта инструкция для безопасности

1. **Нет публичного IP** — сервер недоступен из интернета
2. **Tailscale** — шифрованное соединение между устройствами
3. **Cloud NAT** — только исходящий трафик, входящий заблокирован
4. **Dedicated user** — `openclaw` с ограниченными правами
5. **Права доступа** — 700 на директорию, 600 на конфиги
6. **Pairing mode** — новые пользователи требуют одобрения
7. **Gateway на loopback** — Dashboard недоступен извне

### Рекомендуемые permissions

| Путь | Права |
|------|-------|
| `~/.openclaw/` | 700 |
| `~/.openclaw/openclaw.json` | 600 |
| `~/.openclaw/credentials/*` | 600 |
| `~/.openclaw/agents/*/agent/auth-profiles.json` | 600 |

### Дополнительные рекомендации

- Используйте современные модели (Claude Opus) для агентов с инструментами
- Требуйте mention в публичных группах (`requireMention: true`)
- Отключайте `exec`/`browser`/`web_fetch` для агентов с недоверенным вводом
- Рассмотрите sandbox mode: `agents.defaults.sandbox.mode: "all"`
- Используйте Tailscale Serve вместо публичного доступа

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
| `<CODE>` | Pairing code | Выдаёт бот при первом сообщении |
