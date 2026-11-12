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

По умолчанию порт доступен только на localhost. Публикацию сервиса через
HTTPS/reverse proxy нужно настроить отдельно перед production-деплоем.

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
Secrets, а не в `.env.example`. Перед настройкой CI/CD нужно будет
напомнить добавить их.

Разработка ведётся в `develop`; production-деплой планируется после push
в `main`. CI/CD и необходимые секреты будут настроены отдельно.
