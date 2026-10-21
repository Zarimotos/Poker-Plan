# PRE-04 — первичная настройка существующего GitHub-репозитория

> Эта инструкция предназначена для **уже созданного публичного репозитория GitHub**. В архиве нет `.git`, нет скриптов создания репозитория, и он не меняет настройки аккаунта. Настройки ниже выполняет владелец репозитория самостоятельно.

## 1. Распаковать в локальный клон

1. Склонируйте уже созданный репозиторий: `git clone https://github.com/OWNER/REPO.git`.
2. **Распакуйте содержимое архива в корень клона** — именно туда, где лежит скрытая папка `.git/`. При распаковке не создавайте дополнительную вложенную папку `planning-poker-ready-to-copy/`.
3. Включите отображение скрытых файлов и убедитесь, что скопированы `.github/pull_request_template.md`, `.gitignore` и `.env.example`.
4. Если при создании репозитория вы уже добавили свой README или `.gitignore`, сравните их перед заменой. Никакие файлы чужого проекта перезаписывать не нужно.
5. Не публикуйте реальные учётные данные, токены, закрытые макеты, персональную информацию и документы, для которых не согласована публикация. В том числе проверьте `docs/roadmap.md`: версия 2.2 (15.10.2026) имеет статус **для обсуждения**, а не утверждена.

Для первоначальной загрузки **до настройки защиты веток** откройте терминал в клоне:

```bash
git status
git branch --show-current
git branch -M main
git add .
git commit -m "chore: initialize project repository"
git push -u origin main

git switch -c develop
git push -u origin develop
```

Если `git commit` просит указать личность, задайте свою конфигурацию Git (`git config --global user.name "Your Name"`, `git config --global user.email "you@example.com"`) и повторите команду. Если Git пишет `nothing to commit`, проверьте `git status` и наличие файлов. Если ветка `develop` уже существует, **не пересоздавайте и не перезаписывайте её**, а согласуйте внесение изменений через PR.

После этого удобно оставить локально `develop` активной: `git switch develop`. Рабочие изменения далее идут только через feature-ветки и PR.

## 2. Настроить GitHub

1. **Settings → General → Pull Requests**: включить `Allow squash merging` и `Allow merge commits`; `Allow rebase merging` выключить. По желанию включить `Automatically delete head branches`.
2. **Settings → Branches → Default branch**: после публикации обеих веток назначить `develop` веткой по умолчанию. PR релиза в `main` создаётся с явным выбором `base: main`.
3. **Settings → Branches → Branch protection rules → Add classic branch protection rule**: создать **две отдельные** правила — `main` и `develop`.
4. В каждом правиле включить:
   - `Require a pull request before merging`;
   - `Require approvals` → **1**;
   - `Dismiss stale pull request approvals when new commits are pushed`;
   - `Require approval of the most recent reviewable push` (если доступно);
   - `Require conversation resolution before merging`;
   - `Do not allow bypassing the above settings`;
   - оставить **выключенными** `Allow force pushes` и `Allow deletions`.
5. **Пока не включать** `Require status checks to pass`, поскольку проверок CI в PRE-04 нет. После запуска CI добавьте конкретные существующие проверки.
6. **Не включать** `Require linear history` на `main`: для принятого релиза предусмотрен **merge commit**. В `develop` правило выбора `Squash and merge` пока обеспечивается договорённостью и проверкой ревьюера.

Официальная инструкция GitHub: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule

### Почему два метода объединения включены одновременно

- Задачи `feature/*`, `docs/*`, `fix/*` и т. д. → `develop`: **Squash and merge**.
- После подтверждения релиза `develop` (либо специально подготовленная `release/*`) → `main`: **Create a merge commit**.

В обычных настройках репозитория включение двух способов объединения **не ограничивает автоматически**, какой способ можно применять к конкретной ветке. Проверяйте выбор при слиянии.

## 3. Пригласить команду

`Settings → Collaborators` (или `Collaborators and teams`) → `Add people`.

- Тимлид остаётся владельцем/администратором.
- Бэкенд-разработчику и второму фронтенд-разработчику нужен доступ на запись для рабочих веток, PR и ревью.
- Аналитику — доступ на запись, **если** он сам будет выкладывать документы; дизайнеру — аналогично, если он сам редактирует материалы. В публичном репозитории они и без приглашения могут читать материалы и создавать PR через fork.
- Для репозитория **личного аккаунта** GitHub приглашённые collaborators получают доступ на запись: отдельных ролей `Read`/`Write` как в организации там нет. Более тонкие роли доступны в организации.

## 4. Проверить защиту и готовность процесса

Каждый разработчик выполняет **последовательно**, начиная с актуальной `develop`:

```bash
git switch develop
git pull origin develop
git switch -c docs/verify-yourname
# Добавить в docs/team.md свою строку
git add docs/team.md
git commit -m "docs: verify pull request workflow"
git push -u origin docs/verify-yourname
```

Затем на GitHub: `Compare & pull request` → убедиться, что **base: develop** → заполнить шаблон → получить `Approve` другого разработчика → `Squash and merge` → проверить итог в `develop`. Следующий разработчик начинает после слияния предыдущего PR, чтобы избежать конфликтов в `docs/team.md`.

Проверьте также попыткой отправки **безвредного нового тестового коммита** из отдельной рабочей ветки прямо в `develop` и `main`, что GitHub отклоняет такой push. Не используйте `--force` для тестирования. Защита веток и работа PR подтверждаются **на практике**, а не галочками в настройках.

## 5. Ограничение по праву релиза

Для **публичного репозитория личного аккаунта** GitHub Free возможно требовать PR и ревью, но нельзя штатным ограничением `Restrict who can push to matching branches` выделить только тимлида: эта настройка доступна для определённых **репозиториев организаций**. Сотрудники с правом записи теоретически могут слить одобренный PR в `main`. Поэтому до изменения модели прав решение о релизе и право нажать merge в `main` остаются **организационным соглашением**.

Если нужен технический запрет на merge в `main` всеми, кроме тимлида, рассмотрите перенос репозитория в GitHub Organization и настройку ограничений для `main`, или дополнительную адресную проверку владельца кода. Официальное ограничение: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule

## 6. Кто завершает PRE-04

- **Тимлид:** публикация базовых файлов, `main/develop`, доступы, защита, актуальность дорожной карты, приёмка.
- **Бэкенд/DevOps:** уточняет `.gitignore`, `.env.example`, структуру сервера и предложения по будущему CI.
- **Фронтенд-разработчик 2:** готовит структуру клиента и проходит PR-проверку.
- **Системный аналитик:** переносит актуальную спецификацию, план спринтов, модель данных и API-документы в `docs/`.
- **Дизайнер:** размещает ссылку на актуальную Figma в `docs/design.md` (только если доступ можно публиковать).

**PRE-04 ещё не завершена** после простого копирования файлов. CI/CD, Docker, работающий сервер и полноценный запуск приложения не входят в эту задачу. Пустые `frontend/` и `backend/` не означают собираемое приложение; требование к собираемой `develop` начнёт проверяться после появления сборки.
