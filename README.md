# experiment-github-actions-monorepo

- `+` pros:
  - Швидкий чекаут репозіторію
  - Конфігурація в коді, не вимагає ручного налаштування репозіторія 
  - Не потрібно використовувати додаткові інструменти для створення CI/CD pipeline
  - Нативно відомо про зміни між мерджкоммітами в master, тому легко зробити зборку і деплой лише того, де відбулись зміни
  - Просте створення і публікація власних дій — добре для PR.
  - Кеш просто неймовірний!!!
  - ніяких проблем з лімітами API

- `-` cons:
  - [Investigating - We are investigating reports of degraded performance for GitHub Actions.](https://www.githubstatus.com)
  - [Github Action stuck at queue](https://github.community/t/github-action-stuck-at-queue/16869/139)
  - runner не очищується після роботи джоби.
    Трохи про це і інше https://dev.to/wayofthepie/hacking-together-an-actions-runner-orchestrator-5ef9
    https://github.com/actions-runner-controller/actions-runner-controller
    https://github.com/github-developer/self-hosted-runners-anthos
    Потрібен механізм який би вбивав runner після джоби і створював його знову (runner-controller).
  - Не підтримуються нативний запуск приватних дій
    https://github.com/github/roadmap/issues/74
  - Using cache on tags does not work
    https://github.com/actions/cache/issues/556
  - Якщо запускаємо через workflow_dispatch то:
    - Не інформативний дашбоард workflows по запускам, відсутня можливість додати інформацію (проект, тег, deploy_group, deploy_state)
      https://github.community/t/github-actions-dynamic-name-of-the-workflow-with-workflow-dispatch/150327
    - запускається на чистому бранчі, потрібно чекаутити `refs/pull/{number}/merged` (тільки для відкритого PR).
      Для закритого PR/видаленого бранча, потрібно запускати над бранчем/master, але не втратити інформацію що це дії над PR.
      (для master і tags все як зазвичай)
    - пусті github.event.pull_request, github.event.action, github.base_sha, github.base_ref, потрібно знаходити цю інформацію
    - Для workflow_dispatch не можливо обрати tags в UI
      https://github.community/t/workflow-dispatch-from-a-tag-in-actions-tab/130561
    - не відомо про зміни під час тригеру tags, потрібно витягнути проект з назви тегу (але це нормальна практика)
  - якщо не спрацював Github хук, то складно його відтворити

## Expirience GA in Enterprise

Які є проблеми для впровадження GA?

- В першу чергу, багато нових і не нових функцій не доступні в Enterprise, або не публічних репозіторії
  Це звучить дивно, але це так.
  Наприклад відсутня можливість публікувати GA в приватних репозіторіях для внутрішнього використання
- Та частина GA яка моглаб відповідати за процеси розгортання дуже слабо пророблена
  (Відсутня можливість підписати dipatch_workflow)
  (ранери не очищуються після запуску джоби)


  Take my mony but give me tools.
  Ці складнощі ускладнюють впровадження GA в корпораціях, тому що доводиться шукати обхідні шляхи

- відсутність достатньої інформації для того щоб інтегрувати рани в конвеєр CI/CD
  (ім'я джоби при матричному запуску, jobId або jobUrl)
  доводиться писати власні костилі на JS

- Якщо ви вигружаєте через blablacar/action-download-last-artifact то це займає багато часу, бо на великих рєпо з купою артифактів API починає гальмувати (до хвилини)

- Немає можливості обрати більш потужні машини для операцій, але це можна вирішити селфхостед

- відсутність можливості оновити воркфлоу в всіх рєпо організації

- доступ до артифактів дуже повільний для великих проектів з великою кількістю невеликих артифактів

## Push by a stick

https://github.com/github/roadmap/projects/1?card_filter_query=label%3Aactions

https://github.com/marketplace/actions/paths-changes-filter
https://docs.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions
https://docs.github.com/en/actions/reference/events-that-trigger-workflows

Create a repository dispatch event
https://docs.github.com/en/rest/reference/repos#create-a-repository-dispatch-event

https://github.com/actions/first-interaction
https://github.com/google-github-actions
https://github.com/google-github-actions/get-secretmanager-secrets
https://github.com/github-developer/octochat-gcp


### `workflow_dispatch`

curl -X POST -H 'Accept: application/vnd.github.v3+json' -H 'Authorization: token <secret>' 'https://api.github.com/repos/gillbus/project-stub/actions/workflows/deploy.yml/dispatches' -d '{"ref":"refs/heads/B2B-401.step2","inputs":{"ref":"refs/pull/59/merge","project":".","deploy_group":"dev1","deploy_state":"absent"}}'

curl -X POST \
  -H 'Accept: application/vnd.github.v3+json' \ 
  -H 'Authorization: token <secret>' \
  'https://api.github.com/repos/levonet/experiment-github-actions-monorepo/actions/workflows/runner.yml/dispatches' \
  -d '{"ref":"refs/tags/v0.0.4","inputs":{"deploy_group":"dev2","deploy_state":"balancer"}}'


### `workflow_run`

**Потрібно перевірити таку можливість тригірити workflow по завершенню попереднього**

```yaml
name: Preflight
on:
  - pull_request
  - push
jobs:
  preflight-job:
    runs-on: ubuntu-latest
    steps:
      - run: env
```

```yaml
name: Test
on:
  workflow_run:
    workflows:
    - Preflight
    types:
    - completed
jobs:
  test-job:
    name: Test Step
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          ref: ${{ github.event.workflow_run.head_branch }}
      - run: git branch
      - run: env
```

### `repository_dispatch`

[Tutorial](https://www.r-bloggers.com/2020/07/running-github-actions-sequentially/)

curl -XPOST -u "${{ secrets.PAT_USERNAME}}:${{secrets.PAT_TOKEN}}" -H "Accept: application/vnd.github.everest-preview+json" -H "Content-Type: application/json" https://api.github.com/repos/YOURNAME/APPLICATION_NAME/actions/workflows/build.yaml/dispatches --data '{"ref": "master"}'
on: workflow_dispatch

curl -XPOST -u "${{ secrets.PAT_USERNAME}}:${{secrets.PAT_TOKEN}}" -H "Accept: application/vnd.github.everest-preview+json" -H "Content-Type: application/json" https://api.github.com/repos/YOURNAME/APPLICATION_NAME/dispatches --data '{"event_type": "build_application"}'

curl --location --request GET 'https://api.github.com/repos/levonet/experiment-github-actions-monorepo/actions/workflows/runner.yml/runs?branch=master&per_page=1' -H "Authorization: token <secret>" -H "Content-Type: application/vnd.github.v3+json"

```yaml
name: projects

on: repository_dispatch

jobs:

  server:
    if: github.event.client_payload.job == 'server'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - name: Build Server
        working-directory: ./apps/server
        run: ./gradlew clean build

  client:
      if: github.event.client_payload.job == 'client'
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@master
        - name: Build Client
          working-directory: ./apps/client
          run: ./gradlew clean build
```
