# Автоматизация верификации и развертывания NuGet пакета с помощью GitHub actions

## Цель
На практическом примере настроить CI/DI [GitHub actions](https://github.com/features/actions) для валидации и развертывания NuGet пакета, начиная с минимально рабочего конфигурационного `yaml` файла и постепенно усовершенствовать его до полной автоматизации запланированных требований.

## Table of Contents
* [Окружение, процессы и постановка задачи](#)
* [Первоночальная конфигурация](#)
* [Этапы ручного развертывания](#)
* [Кратко о том, что такое автоматизация с помощью `GitHub actions` и о структуре `yaml` файла](#)
* [1. Создание MVP пайплайна](#)
   * [1.1. Добавление шаблона пайплайна с триггером запуска](#)
   * [1.2. Добавления джобы сборки пакета](#)
   * [1.3. Добавления джобы публикации пакета](#)
   * [1.4. Результат](#)
* [2. Проверка прохождения тестов](#)
* [3. Проверка текущий версии проекта](#)
* [4. Добавление тега с версией на текущий релизный комит](#)
* [5. Создание релиз записи в репозитории GitHub профиля](#)
* [6. Управление зависимостями между джобами и очередностью выполнения](#)
* [Заключение](#)

## Окружение, процессы и постановка задачи

Предположим, мы пишем некую библиотеку, используя стек `C#/.NET` и планируем её в дальнейшем разместить в общий доступ в виде `NuGet` пакета.
На данном этапе у нас есть `.NET` решение (solution) в локальном `Git` репозитории. В качестве удаленного репозитория используем `GitHub` сервис.

Настал этап первого развертывания (deploy), который проведем вручную, чтобы понимать те процессы, которые в последствии будем автоматизировать.
Весь процесс можно условно разделить на две части. Первая часть - это разовая предварительная конфигурация окружения и проекта, вторая - непосредственно ручная рутина для каждой публикации библиотеки.

### Первоночальная конфигурация

1. В качестве хоста `NuGet` пакетов будем использовать [nuget.org](https://www.nuget.org) сервис. Для этого [создадим аккаунт](https://learn.microsoft.com/en-us/nuget/nuget-org/individual-accounts#add-a-new-individual-account), если его еще нет
2. [Генерируем в аккаунте API ключ](https://learn.microsoft.com/en-us/nuget/nuget-org/publish-a-package#create-an-api-key), который нам будет необходим для последующей публикации библиотеки
3. Добавим необходмые [метаданные](https://learn.microsoft.com/en-us/nuget/create-packages/package-authoring-best-practices) в `<PropertyGroup>` и `<ItemGroup>` конфигурационного файла проекта (`*.csproj`) для публикации
```xml
<PropertyGroup>

  <!-- Уникальный индентификатор NuGet пакета и версия -->
  <PackageId>Library.UsefulPackage</PackageId>
  <Version>1.0.1</Version>

  <!-- Информация о лицензии -->
  <PackageLicenseExpression>MIT</PackageLicenseExpression>

  <!-- Авторство, описание проекта, логотип, документация -->
  <Authors>Software Developer</Authors>
  <Title>Краткое описание пакета</Title>
  <Description>Развернутое описание пакета</Description>
  <PackageIcon>logo.png</PackageIcon>
  <PackageReadmeFile>README.md</PackageReadmeFile>
  <GenerateDocumentationFile>True</GenerateDocumentationFile>

  <!-- Ссылка на репозиторий с исходным кодом -->
  <RepositoryUrl>https://github.com/software-developer/useful-library</RepositoryUrl>

  <!-- Теги описывающие проект для индексации и поиска -->
  <PackageTags>dotnet, useful, lib, etc</PackageTags>
</PropertyGroup>

<ItemGroup>

   <!-- Добавление необходмых ссылок для ProprtyGroup элементов -->
   <None Include="..\..\images\logo.png" Pack="true" PackagePath="\"/>
   <None Include="..\..\LICENSE" Pack="true" PackagePath="LICENSE"/>
   <None Include="..\..\README.md" Pack="true" PackagePath="\"/>

</ItemGroup>
```

### Этапы ручного развертывания

1. Запуск юнит тестов
2. Инкрементация [версии](https://learn.microsoft.com/en-us/nuget/create-packages/package-authoring-best-practices#package-version) в проектном конфигурацинном файле: `<Version>1.0.1</Version>`
3. Слияние комитов в мастер ветку и добавление `Git` тега с релизной версией на комит
4. Сборка релизной версии библиотеки: `dotnet pack --configuration Release`
5. Публикации релизной версию на `nuget.org` для общего доступа: `dotnet nuget push {NUGET_NAME_WITH_VERSION} --api-key {API_KEY} --source https://api.nuget.org/v3/index.json`
6. Создание [релиза](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases) в `GitHub` репозитории с описанием изменений в текущей версии для пользователей 

После внесения изменений в нашу библиотеку необходимо раз за разом выполнять перечисленные выше 6 шагов для доставки новой версии пакета конечным пользователем. Возьмем эти пункты в качетсве требований и автоматизируем процесс.

## Кратко о том, что такое автоматизация с помощью `GitHub actions` и о структуре `yaml` файла

`GitHub actions` это инструменты CI/CD для автоматизации рабочих процессов при создании и публикации программного обеспечения. Под рабочими процессами подразумевается управление ветками разработчиков в процессе создания пулриквестов, код ревью и слияния, а так же сборки, тестирования и публикации конечного результата.

Довольно множество операций можно рассматривать как повторяющиеся и рутинные, которые можно автоматизировать и при необходимости добавлять гибкость с помощью параметров или условных конструкций. Другими словами, автоматизировать процесс путем написание скрипта в `yaml` файле, который будет интерпретироваться `GitHUb` сервисов как набор инструкций автоматизации.

`Yaml` файл включает в себя следующие инструкции:
1. Триггеры запуска автоматизации (комит в определенную ветку, пул риквест)
2. Окружение в котором будут выполнятся команды (Тип и версия ОС, контейнер)
3. Описание исполняемых команд. Команды рассматриваются как `steps` в контексте одной `job`. Рабочий процесс `workflow` может содержать несколько `jobs`

Более подробно о том, что из себя представляют `GitHub Actions` можно почитать на [официальном сайте](https://docs.github.com/en/actions/about-github-actions/understanding-github-actions).

Документацию по написанию рабочих процессов можно найти [здесь](https://docs.github.com/en/actions/writing-workflows).

## 1. Создание MVP пайплайна

### 1.1. Добавление шаблона пайплайна с триггером запуска

Добавим в репозиторий проекта файл `./github/workflows/release-and-publish.yml` с указанием имени пайплайна и [события, по которому он будет запускаться](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows). В нашем случае это комит на мастер ветку удаленного репозитория.

```yaml
# Pipeline name
name: Create release and publish NuGet

# Trigger when a commit is created on the remote master branch
on:
  push:
    branches:
      - 'master'

# Сюда будут добавлены джобы в соответствии с требованиями
jobs:
```

Для создания минимально работающей версии пайплайна добавим нам нужно добавить джобу сборки ньюгет пакета и джобу публикации артифакта на nuget.org.

### 1.2. Добавления джобы сборки пакета

Здесь и далее буду показывать только дельту изменения пайплайна. Полную версию можно найти в конце статьи.

```yaml
  # Уникальный референс индентификатор джобы
  create_nuget:
    # Юзер френдли имя джобы, которое будет отображаться на UI
    name: Create NuGet
    # Среда исполения. Каждая джоба выполняется изолировано в своей среде
    runs-on: ubuntu-24.04
    # Save path to the NuGet directory in the environment variable
    env:
      NuGetDirectory: ${{ github.workspace}}/nuget
    # Перечень последовательно запускаемых команд
    steps:
      # Чекаут на комит ветки для доступа к исходному коду
      - name: Checkout repository
        uses: actions/checkout@v4
      
      # Установка SDK 
      - name: Setup .NET
        uses: actions/setup-dotnet@v4

      # Сборка и упаковка пакета
      - name: Pack
        shell: pwsh
        run: dotnet pack .\src\EventLog --configuration Release --output ${{ env.NuGetDirectory }}

      # Загрузка артефакта в хранилище для доступа к нему из других джоб
      - uses: actions/upload-artifact@v4
        with:
          name: nuget
          if-no-files-found: error
          retention-days: 7
          path: ${{ env.NuGetDirectory }}/*.nupkg
```

Как результат имеем загруженный в хранилище собранный артифакт пакет с именем EventLog_1.0.1.nupkg, где версия пакета - это версия прописанная в chproj файле проекта. Необходимо не забывать инкрементить соответствующую часть версии с каждым релизом библиотеки, так как одну и ту же версию не получится опубликовать на nuget.org дважды.

### 1.3. Добавления джобы публикации пакета

```yaml
 deploy:
    name: Deploy NuGet
    runs-on: ubuntu-24.04
    # Перед публикацией необходим готовый артефакт.
    # Поэтому эта джоба ждет завершения выполнения джобы create_nuget
    needs: create_nuget
    # Выполнить, если успешно завершилось выполнение create_nuget
    if: success()
    # Save path to the NuGet directory in the environment variable
    env:
      NuGetDirectory: ${{ github.workspace}}/nuget
    steps:
      # Загружаем содержимое хранилища
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: nuget
          path: ${{ env.NuGetDirectory }}

      - name: Setup .NET Core
        uses: actions/setup-dotnet@v4

      # С помощью dotnet утилиты nuget публикуе пакет
      - name: Publish NuGet package
        shell: pwsh
        run: |
          foreach($file in (Get-ChildItem "${{ env.NuGetDirectory }}" -Recurse -Include *.nupkg)) {
              dotnet nuget push $file --api-key "${{ secrets.NUGET_APIKEY }}" --source https://api.nuget.org/v3/index.json --skip-duplicate
          }
```

Итерируем содержимое загруженного хранилища включая поддиректории и загружаем каждый найденный `nupkg` пакет на `nuget.org`. Чтобы не хранить ключ доступа к хосту ньюгет пакетов в открытом доступе, [сохраняем его приватно в настройках репозитория](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions) и ссылаемся на него из пайплайна.

### 1.4. Результат

По итогу имеем минимально функионирующий пайплайн, не смотря на то, что к нему есть довольно много вопросов таких как:
- Целесщобразность выполнения, если тесты не пройдены успешно
- Защита от не инкрементнутой версии проекта
- Добавление тега на комит релиза
- Создание релиз ноута

Закончим эти пунктиы в последующих разделах.

<img width="600" src="./assets/Pic-1.png"/>

## 2. Проверка прохождения тестов

Чтобы быть уверенным в том, что к публикации не допускается код, все тесты которого не выполнены успешно, добавим соответствующую джобу. 

```yaml
  run_test:
    name: Run tests
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4

      - name: Run tests
        shell: pwsh
        run: dotnet test --configuration Release .\src\EventLog.UnitTests
```

## 3. Проверка текущий версии проекта

В этой джобе мы хотим убедиться, что версия проекта, указанная в `chproj` файле проекта выше последней версии, которую мы извлечем из тега последнего релизного коммита. То есть, что мы не забыли инкрементнуть правильно версию перед пушем кода в удаленный репозиторий.

```yaml
  check_version:
    name: Check project version
    runs-on: ubuntu-24.04
    outputs:
      # Возвращает результат проверки в переменной is_valid
      is_valid: ${{ steps.compare_versions.outputs.is_valid }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Get project version from .csproj
        shell: bash
        run: |
          # Получаем версию проекта из csproj файла
          VERSION=$(grep -oPm1 "(?<=<Version>)[^<]+" ./src/EventLog/EventLog.csproj)
          echo "Project version is $VERSION"
          # Сохраняем результат в переменную VERSION
          echo "VERSION=$VERSION" >> $GITHUB_ENV

      - name: Get latest tag
        id: tag
        run: |
          # Получаем последнюю релизную версию по git тегу из лога репозитория
          git fetch --tags
          LATEST_TAG=$(git tag -l "v*" --sort=-v:refname | head -n 1)
          echo "Latest tag: $LATEST_TAG"
          # Сохраняем результат в переменную LATEST_TAG
          echo "LATEST_TAG=$LATEST_TAG" >> $GITHUB_ENV

      - name: Compare Strings
        id: compare_versions
        run: |
          # Находим максимальную вверсию сравнивая VERSION и LATEST_TAG и сохраняем ее в переменную GREATER_VERSION
          GREATER_VERSION=$(printf "%s\n%s" "$VERSION" "${LATEST_TAG#v}" | sort -V | tail -n 1)
          if [[ "$VERSION" == "$GREATER_VERSION" && "$VERSION" != "${LATEST_TAG#v}" ]]; then
            # Если версия в конфигурацинном файле проекта выше версии тега, то проверка пройдена
            echo "The new release version is ${LATEST_TAG#v}"
            echo "is_valid=true" >> $GITHUB_OUTPUT
          else
            # Иначе сигнализируем об ошибке
            echo "The project version is not incremented"
            echo "is_valid=false" >> $GITHUB_OUTPUT
          fi
```

## 4. Добавление тега с версией на текущий релизный комит

Как минимум иметь git тег с версией релиза нам удобно по 3 причинам:
- В гит логах быстро находить и переключаться на комит соответствующей версии релиза
- На этот тег завязана проверка текущей версии проекта из пункта (3)
- GitHub Release (релиз ноут секция) функционал завязан на соответствующий тег

 ```yaml
  tag_and_push:
    name: Create and push release tag
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # Наконфигурируем в настроцках Git текущей джобы имя и адрес почты автора для тега
      - name: Set up Git
        run: |
          git config --global user.name "${{ secrets.GIT_USER_NAME }}"
          git config --global user.email "${{ secrets.GIT_USER_EMAIL }}"

      # Используя второй раз один и тот же код, неплохо его было бы вынести в отдельный action
      # Для упрощения, оставим дубляж как есть
      - name: Get project version from .csproj
        shell: bash
        run: |
          VERSION=$(grep -oPm1 "(?<=<Version>)[^<]+" ./src/EventLog/EventLog.csproj)
          echo "Project version is $VERSION"
          echo "VERSION=$VERSION" >> $GITHUB_ENV

      - name: Fetch the latest changes from the remote repository
        run: |
          git fetch --tags

      - name: Create and push tag
        run: |
          NEW_TAG="v$VERSION"
          git tag $NEW_TAG
          echo "Tag created: $NEW_TAG"
          git push origin $NEW_TAG
```

## 5. Создание релиз записи в репозитории GitHub профиля

Довольно удобно и информативно, когда пользователи релизного продукта могут зайти в соответствующую секцию и прочитать об изменениях в новой версии и понять, зачем им стоит на нее переходить.

 ```yaml
  release:
    name: Create release
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Create GitHub release
        run: |
          git fetch --tags
          NEW_TAG=$(git describe --tags --abbrev=0 origin/master)
          echo "Latest tag on master: $NEW_TAG"
          gh release create $NEW_TAG \
              --repo="$GITHUB_REPOSITORY" \
              --title="${NEW_TAG#v}" \
              --generate-notes \
              --generate-notes \
              --verify-tag \
              --latest
```

## 6. Управление зависимостями между джобами и очередностью выполнения

В результате автоматизации всех требований мы получили цепочку исполняющихся действий как схематически представлено ниже.

<img width="600" src="./assets/Pic-2.png"/>

Проблема сейчас в том, что джобы выполняются в не зависимости от успешности/не успешности выполнения других джоб, хотя у нас есть явные зависимости и четкая последовательность. К примеру, не имеет смысла выполнять весь воркфлоу, если тесты не пройдены или версия проекта правильно не инкрементнута. Для управления зависимостями и условиями, используются в джобах теги `needs` и `if`, внутри которых можно использовать ссылки на другие джобы или переменные по их уникальным идентификаторам.

По итогу правильно выстроенные зависимости и последовательность исполнения джоб будет выглядеть следующим образом:

<img width="1200" src="./assets/Pic-3.png"/>

Проследить как я этого добился можно по вышеупомянутым тегам в финальном виде `yaml` файла:

```yaml
# Pipeline name
name: Create release and publish NuGet

# Trigger when a commit is created on the remote master branch
on:
  push:
    branches:
      - 'test'

jobs:
  run_test:
    name: Run tests
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4

      - name: Run tests
        shell: pwsh
        run: dotnet test --configuration Release .\src\EventLog.UnitTests

  check_version:
    name: Check project version
    runs-on: ubuntu-24.04
    outputs:
      # Возвращает результат проверки в переменной is_valid
      is_valid: ${{ steps.compare_versions.outputs.is_valid }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Get project version from .csproj
        shell: bash
        run: |
          # Получаем версию проекта из csproj файла
          VERSION=$(grep -oPm1 "(?<=<Version>)[^<]+" ./src/EventLog/EventLog.csproj)
          echo "Project version is $VERSION"
          # Сохраняем результат в переменную VERSION
          echo "VERSION=$VERSION" >> $GITHUB_ENV

      - name: Get latest tag
        id: tag
        run: |
          # Получаем последнюю релизную версию по git тегу из лога репозитория
          git fetch --tags
          LATEST_TAG=$(git tag -l "v*" --sort=-v:refname | head -n 1)
          echo "Latest tag: $LATEST_TAG"
          # Сохраняем результат в переменную LATEST_TAG
          echo "LATEST_TAG=$LATEST_TAG" >> $GITHUB_ENV

      - name: Compare Strings
        id: compare_versions
        run: |
          # Находим максимальную вверсию сравнивая VERSION и LATEST_TAG и сохраняем ее в переменную GREATER_VERSION
          GREATER_VERSION=$(printf "%s\n%s" "$VERSION" "${LATEST_TAG#v}" | sort -V | tail -n 1)
          if [[ "$VERSION" == "$GREATER_VERSION" && "$VERSION" != "${LATEST_TAG#v}" ]]; then
            # Если версия в конфигурацинном файле проекта выше версии тега, то проверка пройдена
            echo "The new release version is ${LATEST_TAG#v}"
            echo "is_valid=true" >> $GITHUB_OUTPUT
          else
            # Иначе сигнализируем об ошибке
            echo "The project version is not incremented"
            echo "is_valid=false" >> $GITHUB_OUTPUT
          fi

  tag_and_push:
     name: Create and push release tag
     runs-on: ubuntu-24.04
     # Джоба ожидает выполнения указанных в условии других джоб
     needs: [run_test, check_version]
     # Создаем  тег после того, как убедились, что юнит тесты пройдены и проверка версии завершена успешно
     if: ${{ success() && needs.check_version.outputs.is_valid == 'true' }}
     steps:
       - name: Checkout repository
         uses: actions/checkout@v4
  
       # Наконфигурируем в настроцках Git текущей джобы имя и адрес почты автора для тега
       - name: Set up Git
         run: |
           git config --global user.name "${{ secrets.GIT_USER_NAME }}"
           git config --global user.email "${{ secrets.GIT_USER_EMAIL }}"
  
       # Используя второй раз один и тот же код, неплохо его было бы вынести в отдельный action
       # Для упрощения, оставим дубляж как есть
       - name: Get project version from .csproj
         shell: bash
         run: |
           VERSION=$(grep -oPm1 "(?<=<Version>)[^<]+" ./src/EventLog/EventLog.csproj)
           echo "Project version is $VERSION"
           echo "VERSION=$VERSION" >> $GITHUB_ENV
  
       - name: Fetch the latest changes from the remote repository
         run: |
           git fetch --tags
  
       - name: Create and push tag
         run: |
           NEW_TAG="v$VERSION"
           git tag $NEW_TAG
           echo "Tag created: $NEW_TAG"
           git push origin $NEW_TAG
  
  release:
     name: Create release
     runs-on: ubuntu-24.04
     needs: tag_and_push
     if: success()
     steps:
       - name: Checkout repository
         uses: actions/checkout@v4
  
       - name: Create GitHub release
         run: |
           git fetch --tags
           NEW_TAG=$(git describe --tags --abbrev=0 origin/master)
           echo "Latest tag on master: $NEW_TAG"
           gh release create $NEW_TAG \
               --repo="$GITHUB_REPOSITORY" \
               --title="${NEW_TAG#v}" \
               --generate-notes \
               --generate-notes \
               --verify-tag \
               --latest
  
  # Уникальный референс индентификатор джобы
  create_nuget:
      # Юзер френдли имя джобы, которое будет отображаться на UI
      name: Create NuGet
      # Среда исполения. Каждая джоба выполняется изолировано в своей среде
      runs-on: ubuntu-24.04
      # Save path to the NuGet directory in the environment variable
      needs: tag_and_push
      if: success()
      env:
        NuGetDirectory: ${{ github.workspace}}/nuget
      # Перечень последовательно запускаемых команд
      steps:
        # Чекаут на комит ветки для доступа к исходному коду
        - name: Checkout repository
          uses: actions/checkout@v4
        
        # Установка SDK 
        - name: Setup .NET
          uses: actions/setup-dotnet@v4
  
        # Сборка и упаковка пакета
        - name: Pack
          shell: pwsh
          run: dotnet pack .\src\EventLog --configuration Release --output ${{ env.NuGetDirectory }}
  
        # Загрузка артефакта в хранилище для доступа к нему из других джоб
        - uses: actions/upload-artifact@v4
          with:
            name: nuget
            if-no-files-found: error
            retention-days: 7
            path: ${{ env.NuGetDirectory }}/*.nupkg
     
  deploy:
      name: Deploy NuGet
      runs-on: ubuntu-24.04
      # Перед публикацией необходим готовый артефакт.
      # Поэтому эта джоба ждет завершения выполнения джобы create_nuget
      needs: create_nuget
      # Выполнить, если успешно завершилось выполнение create_nuget
      if: success()
      # Save path to the NuGet directory in the environment variable
      env:
        NuGetDirectory: ${{ github.workspace}}/nuget
      steps:
        # Загружаем содержимое хранилища
        - name: Download artifact
          uses: actions/download-artifact@v4
          with:
            name: nuget
            path: ${{ env.NuGetDirectory }}
  
        - name: Setup .NET Core
          uses: actions/setup-dotnet@v4
  
        # С помощью dotnet утилиты nuget публикуе пакет
        - name: Publish NuGet package
          shell: pwsh
          run: |
            foreach($file in (Get-ChildItem "${{ env.NuGetDirectory }}" -Recurse -Include *.nupkg)) {
                dotnet nuget push $file --api-key "${{ secrets.NUGET_APIKEY }}" --source https://api.nuget.org/v3/index.json --skip-duplicate
            }
```

## Заключение

В этой статье я максимально сжато показал свой опыт написания пайплайна для GitHub Actions автоматизирая рутинные действия по публикации NuGet проекта, что позволяет больше фокусироваться на функционале, не производя каждый раз единообразные манипуляции по публикации релизов. Слияние нового функционала с проставленной версией в мастер ветку приводит к автоматизированным этапам проверки и публикации NuGet пакета на nuget.org, что делает его через короткое время доступным для скачивания через NuGetPackage Manager или любой другой клиент NuGet пакетов.

Так же после выполнения пайплайна не стоит забывать обновить соответствующую секцию релиза в GitHub профилe.

[How to create github actions workflow for the handling nuget package development and delivery](EN.md)
