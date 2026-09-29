# Деплой через Jenkins

Сайт статический (index.html + redesign.css + assets), на сервере `ta.commit.kz`
его раздаёт nginx из `/var/www/tolyqadam`. Основной репозиторий —
`almau-it/tolyqadam` (форк исходного `kuanyshtimuruly/tolyqadam`).
Пайплайн выполняется на агенте **sites2** — он стоит на самом веб-сервере,
поэтому деплой локальный, без SSH: Jenkins запускает `update.sh`
(git pull → chown/chmod → reload nginx).

## Требования

1. Плагины Jenkins: **Pipeline**, **Git** (обычно уже установлены).
2. Агент с меткой `sites2` подключён и онлайн (*Manage Jenkins → Nodes*).
   Если нода называется `sites2`, но метки у неё нет — добавьте `sites2`
   в поле Labels.
3. `update.sh` требует прав root (`chown root:www-data`, `systemctl reload nginx`).
   Если агент sites2 работает не от root, дайте его пользователю sudo без пароля
   на этот скрипт:

   ```
   # /etc/sudoers.d/jenkins-tolyqadam
   jenkins ALL=(root) NOPASSWD: /var/www/tolyqadam/update.sh
   ```

   и поменяйте в Jenkinsfile команду деплоя на `sudo ${DEPLOY_PATH}/update.sh`.
4. Клон на сервере (`/var/www/tolyqadam`) должен тянуть из форка, иначе
   `git pull` продолжит забирать старый репозиторий:

   ```
   cd /var/www/tolyqadam
   git remote set-url origin https://github.com/almau-it/tolyqadam.git
   ```

## Создание джобы

Вариант A — **Multibranch Pipeline** (рекомендуется):

1. *New Item → Multibranch Pipeline*
2. Branch Sources → Git → URL: `https://github.com/almau-it/tolyqadam.git`
3. Build Configuration: *by Jenkinsfile*, путь `Jenkinsfile`
4. Деплой выполняется только для ветки `main` (условие `when { branch 'main' }`),
   остальные ветки проходят только проверку файлов.

Вариант B — обычный **Pipeline**:

1. *New Item → Pipeline*
2. Definition: *Pipeline script from SCM*, SCM: Git, тот же URL, ветка `*/main`
3. Script Path: `Jenkinsfile`

## Автозапуск по пушу

- Если Jenkins доступен из интернета: в настройках репозитория
  `almau-it/tolyqadam` на GitHub добавьте webhook
  `https://<jenkins-host>/github-webhook/` (плагин **GitHub** должен быть
  установлен) — сборка будет стартовать сразу после пуша.
- Если нет — включите в джобе *Poll SCM*, например `H/5 * * * *`
  (проверка изменений каждые 5 минут).

## Что делает пайплайн

| Stage  | Действие |
|--------|----------|
| Verify | Проверяет, что `index.html`, `redesign.css`, `assets/` и `update.sh` на месте |
| Deploy | Только для `main`: запускает `/var/www/tolyqadam/update.sh` прямо на агенте sites2 |

## Переход с GitHub Actions

Унаследованный из исходного репозитория workflow `.github/workflows/deploy.yml`
делает то же самое по SSH. В форке он не активен (Actions в форках выключены
по умолчанию, и секрета `SSH_PRIVATE_KEY` здесь нет), но чтобы не путал —
удалите файл, когда Jenkins заработает.
