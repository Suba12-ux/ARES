# ARES

Минимальный FastAPI-каркас персонального голосового ассистента. На этом
этапе реализован только health endpoint; интеграции LLM, STT и TTS ещё
не подключены.

## Локальный запуск

Скопируйте `.env.example` в `.env`. Файл `.env` не следует добавлять в
Git. Для текущего health endpoint API-ключи не нужны.

В PowerShell из корня репозитория:

```powershell
Copy-Item .env.example .env
docker compose up --build
```

Проверка:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/health
```

По умолчанию порт доступен только на localhost. HTTPS/reverse proxy
настраивается отдельно; автоматизация ниже не меняет существующую
серверную конфигурацию и не включает TLS.

## Тесты

```powershell
Set-Location backend
python -m pip install -r requirements-dev.txt
python -m pytest
```

Для запуска без Docker из каталога `backend`:

```powershell
python -m pip install -r requirements.txt
python -m uvicorn app.main:app --reload
```

## Переменные окружения

Список имён находится в `.env.example`. `DEEPSEEK_API_KEY` понадобится
при подключении LLM. Переменные `STT_*` и `TTS_*` пока общие
заполнители: конкретные провайдеры и их конфигурация не выбраны. Не
вставляйте реальные ключи в документацию или Git.

Секреты деплоя для CI/CD будут задаваться отдельно в GitHub Actions
Secrets и Variables в окружении `production`, а не в `.env.example`.

## CI/CD и деплой

Workflow `.github/workflows/ci-cd.yml` запускает тесты и сборку Docker
образа на push и pull request в `develop` и `main`. Только успешный push
в `main` публикует образ `ghcr.io/suba12-ux/ares:<commit-sha>` в GHCR и
обновляет сервер через SSH. Публикуемый образ поддерживает Linux
`amd64` и `arm64`. При неуспешной проверке health check
workflow пытается вернуть предыдущий образ. Локально `docker compose`
объединяет `compose.yaml` с `compose.override.yaml`, который добавляет
сборку backend из исходников. На сервер workflow передаёт только
`compose.yaml` и явно использует его без локального override.

Перед включением deploy job создайте GitHub Actions Environment с именем
`production` и добавьте в нём следующие **Variables**:

| Variable | Значение |
| --- | --- |
| `DEPLOY_HOST` | `109.172.7.1` |
| `DEPLOY_USER` | SSH-пользователь с правом запускать Docker Compose |
| `DEPLOY_PATH` | Абсолютный путь к существующему каталогу приложения на сервере; там должен находиться серверный `.env` |
| `DEPLOY_PORT` | SSH-порт, по умолчанию `22` |

Добавьте в то же окружение следующие **Secrets**:

| Secret | Значение |
| --- | --- |
| `DEPLOY_SSH_KEY` | Приватный SSH-ключ без парольной фразы; соответствующий публичный ключ должен быть разрешён на сервере |
| `DEPLOY_KNOWN_HOSTS` | Полная проверенная строка `known_hosts`: host, тип ключа и полный публичный ключ сервера |

Пользователь деплоя должен иметь доступ к Docker без интерактивного
ввода пароля. На сервере должны быть установлены Docker Engine и Docker
Compose plugin версии с поддержкой `docker compose up --wait`. Разрешите
`GITHUB_TOKEN` публикацию пакетов в настройках репозитория, если она
ограничена. Путь `DEPLOY_PATH` должен соответствовать каталогу
текущего развертывания: workflow заменяет там `compose.yaml`, но
сохраняет существующий `.env`; не используйте каталог с несвязанными
Compose-сервисами. Откройте SSH-порт для GitHub Actions runners.

Получите host key по доверенному каналу и проверьте его fingerprint
перед добавлением в `DEPLOY_KNOWN_HOSTS`. На сервере выполните:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
cat /etc/ssh/ssh_host_ed25519_key.pub
```

Скопируйте полную строку `.pub` и добавьте в начало `109.172.7.1 `.
Для порта 22 получится формат
`109.172.7.1 ssh-ed25519 <полный-публичный-ключ> [комментарий]`.
Комментарий необязателен. Для нестандартного порта host записывается как
`[host]:port`. Не используйте вместо полного ключа только тип
`ssh-ed25519`; fingerprint сверьте по доверенному каналу.

Сервер скачивает образ анонимно, поэтому GHCR package `ares` должен быть
публичным. Сначала первый push в `main` опубликует пакет, но deploy может
завершиться ошибкой, пока пакет приватный. Затем в настройках пакета
GitHub Packages смените видимость на Public и повторно запустите
упавший deploy job из Actions. После этого последующие push в `main`
будут деплоиться автоматически без `GHCR_PULL_TOKEN`. Учитывайте, что
публичный образ может скачать любой пользователь.

HTTPS пока не настраивается; доступ к приложению и его портам
определяется уже существующей серверной конфигурацией.
