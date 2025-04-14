# Автоматизация верификации и развертывания NuGet пакета с помощью GitHub action

## Цель
На практическом примере настроить CI/DI [GitHub actions](https://github.com/features/actions) для валидации и развертывания NuGet пакета, начиная с минимально рабочего конфигурационного `yaml` файла и постепенно усовершенствуя его до покрытия всех запланированных требований.

## Table of Contents
* [Кратко об окружении: гит, гит хаб и инструменты CI/CD, nuget.org](#)
* [Начало](#)
* [Развертывание пакета вручную](#)

## Кратко об окружении, процессах и постановка задачи

Предположим, мы пишем некую библиотеку, используя стек `C#/.NET` и планируем её в дальнейшем разместить в общий доступ в виде `NuGet` пакета.
На данном этапе у нас есть `.NET` решение (solution) в локальном `Git` репозитории. В качестве удаленного репозитория используем `GitHub` сервис.

Настал этап первого развертывания (deploy), который проведем вручную, чтобы понимать те процессы, которые в последствии будем автоматизировать.
Весь процесс можно условно разделить на две части. Первая часть - это разовая предварительная конфигурация окружения и проекта, вторая - непосредственно ручная рутина для каждой публикации библиотеки.

Первоночальная разовая конфигурация:
1. В качестве хоста NuGet пакетов будем использовать [nuget.org](https://www.nuget.org) сервис. Для этого [создадим аккаунт](https://learn.microsoft.com/en-us/nuget/nuget-org/individual-accounts#add-a-new-individual-account), если его еще нет
2. [Генерируем в аккаунте API ключ](https://learn.microsoft.com/en-us/nuget/nuget-org/publish-a-package#create-an-api-key), который нам будет необходим для последующей публикации библиотеки
3. Добавим необходмые [метаданные](https://learn.microsoft.com/en-us/nuget/create-packages/package-authoring-best-practices) в `<PropertyGroup>` конфигурационного файла проекта (`*.csproj`) для публикации:
   1. `<PackageLicenseExpression>MIT</PackageLicenseExpression>` - информация о лицензии. В нашем случаеб наиболее популряная для open-source проектов [MIT](https://gist.github.com/nicolasdao/a7adda51f2f185e8d2700e1573d8a633#1-mit)
   2. `<PackageId>Software.UsefulLibrary</PackageId>` - уникальный индентификатор NuGet пакета. Другими словами слот или адрес, по которому можно будет в дальнейшем ссылаться на опубликованный пакет
   4. `<Authors>Software Developer</Authors>`
   5. `<Title>I'm title</Title>`
   6. `<Description>I'm description</Description>`
   7. `<PackageIcon>i_am_icon.png</PackageIcon>`
   8. `<PackageReadmeFile>README.md</PackageReadmeFile>`
   9. `<RepositoryUrl>https://github.com/cat-begemot/event-log</RepositoryUrl>`
   10. `<PackageTags>dotnet, useful, lib, etc</PackageTags>`

Этапы ручного развертывания:

1. Запуск юнит тестов
2. Инкрементация [версии](https://learn.microsoft.com/en-us/nuget/create-packages/package-authoring-best-practices#package-version) в проектном конфигурацинном файле: `<Version>1.0.1</Version>`
3. Слияние комитов в мастер ветку и добавление `Git` тега с релизной версией на комит
4. Сборка релизной версии библиотеки: `dotnet pack --configuration Release`
5. Публикации релизной версию на `nuget.org` для общего доступа: `dotnet nuget push {NUGET_NAME_WITH_VERSION} --api-key {API_KEY} --source https://api.nuget.org/v3/index.json`
6. Создание [релиза](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases) в `GitHub` репозитории с описанием изменений в текущей версии для пользователей 



## Кратко об окружении: гит, гит хаб и инструменты CI/CD, nuget.org

## 1. Начало. Шаблонное приложение либы, связывание с ремоут гитхаб репозиторием

## 2. Развертывание пакета вручную

## 3. Описание процесса создание и улучшения пайплайна

## 4. Чтобы что-то задеплоить, надо иметь собранный пакет. Начальный пайплайн должен состоять минимум из двух джоб

## 5. Так как мы что-то релизим, то добавим джобу для создания релиз ноты на гитхабе

## 6. Создание тега на комите релиза с версией ньюгета

## 7. Добавление проверки версии сборки

## 8. Перед релизом было бы хорошо убедиться, что юнит тесты успешно выполняются и финальный результат

## Вывод: мы имеем настроенный энвайромент со следущим форкфлоу

[How to create github actions workflow for the handling nuget package development and delivery](EN.md)
