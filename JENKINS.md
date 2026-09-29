# Деплой через Jenkins

Сайт статический (index.html + redesign.css + assets). Основной репозиторий —
`almau-it/tolyqadam` (форк исходного `kuanyshtimuruly/tolyqadam`).

Схема деплоя: пайплайн выполняется на агенте **sites2**, который стоит на самом
веб-сервере. Jenkins делает checkout в свой workspace
(`/home/jenkins/agent/workspace/tolyqadam`), а nginx
(`tolyqadam.almau.edu.kz`) раздаёт сайт прямо из этого каталога. Отдельного
шага копирования нет: успешный checkout — это и есть деплой, стадия Publish
только выставляет права на чтение. Перезагрузка nginx не нужна.

Скрипт `update.sh` к этой схеме отношения не имеет — он остался от старого
деплоя через GitHub Actions в `/var/www/tolyqadam`.

## Требования

1. Плагины Jenkins: **Pipeline**, **Git** (обычно уже установлены).
2. Агент с меткой `sites2` подключён и онлайн (*Manage Jenkins → Nodes*).
3. На машине агента установлен git: `sudo apt install -y git`
   (без него падает уже checkout: «Cannot run program "git"»).
4. nginx (`www-data`) может пройти в каталог workspace — домашние каталоги
   по умолчанию закрыты:

   ```
   sudo chmod o+x /home/jenkins /home/jenkins/agent /home/jenkins/agent/workspace
   ```

## Nginx

В server-блоке `tolyqadam.almau.edu.kz`:

```nginx
# root — это КАТАЛОГ, без /index.html на конце
root /home/jenkins/agent/workspace/tolyqadam;
index index.html;

# workspace — git-клон: закрыть служебный каталог от внешнего доступа
location ~ /\.git {
    deny all;
}
```

После правки: `sudo nginx -t && sudo systemctl reload nginx`.

## Создание джобы

Обычный **Pipeline** (не Multibranch — иначе изменится путь workspace,
на который смотрит nginx):

1. *New Item → Pipeline*, имя `tolyqadam` (имя = каталог workspace,
   при другом имени поправьте `root` в nginx)
2. Definition: *Pipeline script from SCM*, SCM: Git,
   URL `https://github.com/almau-it/tolyqadam.git`, ветка `*/main`
3. Script Path: `Jenkinsfile`

## Автозапуск по пушу

- Если Jenkins доступен из интернета: в настройках репозитория
  `almau-it/tolyqadam` на GitHub добавьте webhook
  `https://<jenkins-host>/github-webhook/` (плагин **GitHub** должен быть
  установлен) — сборка будет стартовать сразу после пуша.
- Если нет — включите в джобе *Poll SCM*, например `H/5 * * * *`
  (проверка изменений каждые 5 минут).

## Что делает пайплайн

| Stage   | Действие |
|---------|----------|
| Verify  | Проверяет, что `index.html`, `redesign.css` и `assets/` на месте |
| Publish | `chmod -R a+rX` на workspace, чтобы файлы были читаемы для nginx |

Если сборка упала, workspace не очищается — nginx продолжает раздавать
предыдущую версию сайта.

## Наследие GitHub Actions

Workflow `.github/workflows/deploy.yml` — старый деплой по SSH на
`ta.commit.kz`. В форке он не активен (Actions в форках выключены по
умолчанию, секрета `SSH_PRIVATE_KEY` нет); когда Jenkins заработает,
файл можно удалить вместе с `update.sh`.
