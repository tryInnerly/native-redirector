# Деплой на сервер с Caddy через GitHub

Пошаговая инструкция для развёртывания web-redirector на Linux-сервере с автоматическим деплоем через `git pull`.

## Требования

- Сервер с установленным Caddy и существующим Caddyfile
- Домен, направленный на IP сервера (A-запись в DNS)
- SSH-доступ к серверу

## 1. Клонирование репозитория

### 1.1. Создать SSH-ключ для GitHub

Если на сервере уже есть ключ, привязанный к другому репозиторию, нужно создать отдельный:

```bash
ssh-keygen -t ed25519 -C "native-redirector-deploy" -f ~/.ssh/deploy_native_redirector -N ""
```

Добавить алиас в SSH-конфиг:

```bash
cat >> ~/.ssh/config << 'EOF'
Host github-redirector
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_native_redirector
EOF
```

Посмотреть публичный ключ и добавить его в GitHub как deploy key (**Settings** → **Deploy keys** → **Add deploy key**):

```bash
cat ~/.ssh/deploy_native_redirector.pub
```

### 1.2. Клонировать

```bash
sudo chown $USER:$USER /var/www
cd /var/www
git clone github-redirector:osomov/native-redirector.git
```

## 2. Добавление сайта в Caddyfile

Открыть существующий конфиг:

```bash
sudo nano /etc/caddy/Caddyfile
```

Добавить блок в конец файла (заменить `your-domain.com` на свой домен):

```caddyfile
redirect.tryinnerly.com {
    root * /var/www/native-redirector/redirect
    file_server

    try_files {path} /index.html

    header {
        Cache-Control "public, max-age=3600"
    }

    encode gzip zstd
}
```

> Caddy автоматически получит SSL-сертификат от Let's Encrypt для нового домена.

Применить конфиг:

```bash
sudo systemctl reload caddy
```

Проверить:

```bash
curl -I https://your-domain.com
```

## 3. Автоматический деплой через GitHub Actions

Деплой работает через `.github/workflows/deploy.yml` — при пуше в `main` GitHub подключается к серверу по SSH и делает `git pull`.

### 3.1. Создать SSH-ключ для деплоя

На сервере:

```bash
ssh-keygen -t ed25519 -C "deploy" -f ~/.ssh/deploy_key -N ""
cat ~/.ssh/deploy_key.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/deploy_key
```

Скопировать приватный ключ (вывод последней команды) — он понадобится в следующем шаге.

### 3.2. Добавить секреты в GitHub

Открыть репозиторий на GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

Добавить три секрета:

| Имя | Значение |
|-----|----------|
| `SSH_HOST` | IP-адрес или домен сервера |
| `SSH_USER` | SSH-пользователь (например `root` или `deploy`) |
| `SSH_PRIVATE_KEY` | Приватный ключ из шага 3.1 (целиком, включая `-----BEGIN/END-----`) |

### 3.3. Проверка

1. Сделать любой коммит и пуш в `main`:
   ```bash
   git add . && git commit -m "test deploy" && git push
   ```
2. Открыть **Actions** в репозитории на GitHub — должен появиться запуск «Deploy»
3. Открыть `https://your-domain.com` в браузере

## Альтернатива: ручной деплой

Если автоматический деплой не нужен, достаточно шагов 1–2. Для обновления — заходить на сервер и делать:

```bash
cd /var/www/native-redirector && git pull origin main
```
