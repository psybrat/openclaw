# Установка OpenClaw в изолированном контуре (air-gapped)

Инструкция по установке OpenClaw из предварительно подготовленного бандла внутри контура **без доступа к интернету** (GitHub, npmjs.com, PyPI недоступны).

## Требования

| Компонент | Версия |
|-----------|--------|
| Node.js   | 22+    |
| pnpm      | 10+    |
| npm       | любая  |

## Структура бандла

```
openclaw-bundle/
├── openclaw/                   # Исходный код + node_modules + .pnpm-store
│   ├── .npmrc                  # store-dir=./.pnpm-store
│   ├── .pnpm-store/            # Локальный кеш всех npm-пакетов
│   ├── node_modules/           # Установленные зависимости
│   ├── package.json
│   ├── pnpm-lock.yaml
│   └── src/
└── plugins/                    # (опционально) .tgz файлы плагинов
    ├── openclaw-matrix-*.tgz
    └── openclaw-voice-call-*.tgz
```

## Установка

### 1. Перенести архив в контур

Скопировать `openclaw-bundle.tar.gz` на целевую машину любым доступным способом (USB, SCP через промежуточный хост и т.д.).

### 2. Распаковать

```bash
cd ~
tar xzf openclaw-bundle.tar.gz
cd openclaw-bundle
```

### 3. Настроить pnpm

```bash
pnpm setup
source ~/.bashrc
```

Если `pnpm setup` не срабатывает, задайте путь вручную:

```bash
export PNPM_HOME="$HOME/.local/share/pnpm"
mkdir -p "$PNPM_HOME"
export PATH="$PNPM_HOME:$PATH"

# Сохранить в профиль
echo 'export PNPM_HOME="$HOME/.local/share/pnpm"' >> ~/.bashrc
echo 'export PATH="$PNPM_HOME:$PATH"' >> ~/.bashrc
```

### 4. Установить зависимости в offline-режиме

```bash
cd openclaw

# Проверить, что .npmrc указывает на локальный store
cat .npmrc
# Ожидаемый вывод: store-dir=./.pnpm-store

# Установить зависимости из локального store (без обращения к реестру)
pnpm install --offline --ignore-scripts
```

> **Зачем `--ignore-scripts`?** Пакет `node-llama-cpp` в postinstall пытается скачать cmake и скомпилировать нативный бинарник — в контуре без интернета это упадёт. Флаг `--ignore-scripts` пропускает такие скрипты. Прекомпилированный бинарник уже включён в бандл.

### 5. Собрать проект

```bash
pnpm ui:build
pnpm build
```

### 6. Установить глобально

```bash
pnpm link --global
```

Проверить:

```bash
openclaw --version
```

### 7. Установить плагины (опционально)

```bash
openclaw plugins install ~/openclaw-bundle/plugins/openclaw-matrix-*.tgz
openclaw plugins install ~/openclaw-bundle/plugins/openclaw-voice-call-*.tgz
```

### 8. Первичная настройка

```bash
openclaw onboard
```

Мастер настройки спросит:
- Имя ассистента
- LLM-провайдер (выбрать Ollama или локальный эндпоинт)
- API-ключ (для локальных моделей может не потребоваться)
- Каналы связи (Telegram, Slack и т.д.)

### 9. Запуск

```bash
# Установить и запустить daemon
openclaw daemon install
openclaw daemon start

# Проверить статус
openclaw gateway status
```

## Устранение неполадок

### `ERR_PNPM_NO_GLOBAL_BIN_DIR`

Не настроена директория для глобальных бинарников pnpm. Выполните шаг 3 (настройка pnpm).

### `pnpm install` падает на `node-llama-cpp`

Убедитесь, что используете флаг `--ignore-scripts`:

```bash
pnpm install --offline --ignore-scripts
```

Если `node-llama-cpp` нужен с компиляцией (для прямого запуска LLM без отдельного сервера), установите инструменты сборки:

```bash
# CentOS / RHEL
dnf groupinstall -y "Development Tools"
dnf install -y cmake

# Ubuntu / Debian
apt install -y build-essential cmake
```

И запустите установку без `--ignore-scripts`:

```bash
pnpm install --offline
```

### `pnpm install --offline` не находит пакеты

Убедитесь, что `.npmrc` содержит `store-dir=./.pnpm-store` и директория `.pnpm-store` присутствует в распакованном архиве. pnpm store должен находиться на том же разделе диска, что и проект.

## Важные замечания

- **LLM-провайдер:** В контуре без интернета облачные API (OpenAI, Anthropic) недоступны. Используйте Ollama или llama.cpp. Файлы моделей (GGUF) также необходимо перенести в контур заранее.
- **Архитектура:** Бандл привязан к архитектуре и ОС машины сборки (x64/arm64, Linux/macOS). Целевая машина должна совпадать.
- **Skills:** Навыки из ClawHub — это директории с файлом `SKILL.md`. Скачайте заранее и поместите в `~/.openclaw/skills/`.
