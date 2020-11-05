# Fastlane Remote Config + Gitlab CI/CD

Репозиторий содержит конфигурацию сборщика мобильных приложений, которую можно использовать без включения в сами проекты мобильных приложений, а также запускать на различных CI/CD. В данном случае будет рассматриваться использование в Gitlab CI/CD.

## Агент

Физическая машина, на которой запущен Gitlab Runner, настроенный на выполнение определенных задач CI.

## Авторизация в учетной записи Apple

Для осуществления сборки iOS приложений, по мимо указания общих сведений в конфигурации проекта, требуется авторизация в учётной записи владельца apple_id. Авторизацию нужно выполнять через Fastlane CredentialsManager, используя двухфакторную аутентификацию, чтобы не хранить эти данные в открытом доступе. Для того, чтобы это сделать, нужно выполнить несколько команд на агенте (потребуется ручной ввод пароля):

- Сгенерировать apple app-specific password (https://support.apple.com/en-us/HT204397)
- Добавить учетные данные владельца apple_id в keychain

```bash
$ fastlane fastlane-credentials add --username <apple_id>
```

- Сгенерировать сессионный ключ

```bash
$ fastlane spaceauth -u <apple_id>
```

- Сохранить, полученные пароль и ключ в закрытых переменных проекта в Gitlab (https://docs.gitlab.com/ee/ci/variables/#mask-a-custom-variable)

```bash
FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD - сгенерированный пароль
FASTLANE_SESSION - сгенирированный сессионный ключ
```

> **WARNING**: FASTLANE_SESSION выписывается на 1 месяц, поэтому его нужно обновлять вручную.

## Конфигурация Fastlane в проекте

Чтобы агент мог вызывать команды сборщика внутри проекта, используя общую конфигурацию, нужно добавить локальную конфигурацию Fastlane по шаблону. В корне проекта мобильного приложения нужно создать папку fastlane и скопировать в нее файлы конфигурации из репозитория https://github.com/quarrant/fastlane-remote-config/tree/master/app-config и проставить свои значения переменных.

> **Appfile** - общие переменные проекта. **Matchfile** - конфигурация для работы с сертификатами (iOS), часть переменных загружается из **Appfile**, чтобы избежать дублирования. **Fastfile** - точка входа для сборщика, содержит только импорт конфигурации из общего репозитория.

## Подпись приложений

### Сертификаты (iOS)

https://github.com/quarrant/fastlane-remote-config/blob/master/app-config/Matchfile#L1 указывает на репозиторий для хранения сгенерированных сертификатов. Это должен быть приватный репозиторий.

Для упрощения работы с получением и обновлением сертификатов применяется механизм fastlane match. Что это значит? Сертификаты выписываются 1 раз и сохраняются в репозитории. Так как проект настраивается на сертификаты выписанные через fastlane match, то и разработчики должны выписывать себе сертификаты через этот механизм.

**Генерация новых сертификатов**

Перед началом работы с fastlane match нужно создать сертификаты в репозитории. Делать это можно на любом устройстве, имеющем доступ к учетным данным владельца apple_id. Переходим в корень проекта и выполняем команду:

```bash
$ fastlane match development # создает сертификат разработчика
$ fastlane match appstore # создает сертификат для релизной сборки
```

После этого, созданные сертификаты добавятся в Keychain и запишутся в репозиторий, в ветку соответствующую team_id, указанному в Appfile. Если сертификаты уже есть в репозитории, то они просто будут добавлены в Keychain.

> Чтобы пересоздать сертификаты нужно использовать флаг --force

**Загрузка существующих сертификатов**

Чтобы загрузить себе сертификат разработчика нужно в корне проекта выполнить команду:

```bash
$ fastlane match development --readonly
```

После ее выполнения сертификат и provision будут добавлены в Keychain.

> При указании флага --readonly сертификаты будут просто загружены, при их наличии в репозитории. Отсутствующие сертификаты создаваться не будут.

### Keystore (Android)

Все параметры, которые были указаны при первоначальной генерации keystore должны быть указаны в закрытых переменных Gitlab (https://docs.gitlab.com/ee/ci/variables/#mask-a-custom-variable), в том числе и сам keystore, закодированный в base64.

- Сгенерировать релизный keystore можно следуя инструкции в официальной документации (https://developer.android.com/studio/publish/app-signing#generate-key)
- Добавить закрытые переменные проекта

```bash
ANDROID_RELEASE_KEY_ALIAS - псевдоним для key
ANDROID_RELEASE_KEY_PASSWORD - указанный пароль для key
ANDROID_RELEASE_STORE_PASSWORD - указанный пароль для store
ANDROID_RELEASE_KEYSTORE - закодированное содержимое сгенерированного файла *.keystore
```

## Конфигурация Gitlab CI

Конфигурация задач для выполнения агентом описывается в файле .gitlab-ci.yml.

### Анкеры

В анкерах описываются все общие части задач, чтобы избежать ненужного дублирования.

### Тэги

Данные тэги указывают, что задача должна выполняться на агенте.

```yaml
.mobile-tags: &mobile-tags
  tags:
    - macos
    - mobile
```

### Кэширование зависимостей

Так как node_modules и Pods используют lock-файлы для фиксации установленных версий зависимостей, то эти каталоги можно и нужно кэшировать, чтобы сократить время выполнения задачи.

```yaml
.mobile-cache: &mobile-cache
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - mobile/node_modules
      - mobile/ios/Pods
```

### Переменные

Здесь указываются переменные, которые Fastlane будет считывать из Environments

```yaml
.mobile-build-vars: &mobile-build-vars
  variables:
    GIT_STRATEGY: fetch # Получение репозитория на агенте. Репозиторий будет склонирован в первый раз и дальше будет только забирать изменения. Перед каждым запуском задачи не отслеживаемые файлы будут удаляться.
    ANDROID_BUILD_OUTPUT: ${CI_PROJECT_DIR}/fastlane/build
    ANDROID_PROJECT_DIRECTORY: ${CI_PROJECT_DIR}/android
    ANDROID_RELEASE_KEYSTORE_FILE: ${CI_PROJECT_DIR}/android/release.keystore
    IOS_BUILD_OUTPUT: ${CI_PROJECT_DIR}/fastlane/build
    IOS_PROJECT_DIRECTORY: ${CI_PROJECT_DIR}/ios
    IOS_PROJECT_FILE: ${CI_PROJECT_DIR}/ios/CustomerApp.xcodeproj
    IOS_PROJECT_WORKSPACE: ${CI_PROJECT_DIR}/ios/CustomerApp.xcworkspace
    IOS_BUILD_SCHEME: CustomerApp
    IOS_IPA_NAME: CustomerApp.ipa
```

### Секция script

Также можно вынести части скриптов, выполняющих установку зависимостей и различные генерации.

```yaml
.mobile-prepare-script: &mobile-prepare-script |-
  yarn install
  yarn podinstall
  yarn generate:imagesets
  yarn generate:grpc-web-service
  echo $ANDROID_RELEASE_KEYSTORE | base64 -D > ${ANDROID_RELEASE_KEYSTORE_FILE}
```

### Артефакты

Они нужны для того, чтобы сохранить результат выполнения одной задачи и использовать его в другой, например, собранный файл приложения, который нужно опубликовать.

```yaml
artifacts:
  paths:
    - build/App.*
```

### Ограничитель запуска CI

Указывает, что сборка может запускаться с пуша в master-ветку или с тэга.

```yaml
.master-tags-only: &master-tags-only
  only:
    refs:
      - tags
      - master
```

### Общие правила запуска CI

Нужны для того, чтобы запуск джоб осуществлялся только на создании тэга, а не на push в ветку

```yaml
workflow:
  rules:
    - if: $CI_COMMIT_TAG
      when: always
    - when: never
```

### Пример конфигурации

Пример конфигурации для выполнения сборки и публикации iOS и Android. Запуск сборки осуществляется по тэгу, созданному с master-ветки. В имени тэга указывается версия приложения и вариант сборки. Эти данные разбираются fastlane на составляющие и проставляются, как переменные в сборку.

- **2.1.8-beta.1** // запустит сборку beta конфигурации и проставит версию 2.1.8
- **2.1.8-release.1** // запустит сборку release конфигурации и проставит версию 2.1.8

```yaml
build:
  stage: build
  <<: *master-tags-only
  <<: *mobile-tags
  <<: *mobile-build-vars
  <<: *mobile-cache
  <<: *mobile-before-script
  script:
    - *mobile-prepare-script
    - fastlane android build
    - fastlane ios build
    - mv ${ANDROID_BUILD_OUTPUT}/* ../
    - mv ${IOS_BUILD_OUTPUT}/* ../
  artifacts:
    paths:
      - app-release.*
      - CustomerApp.*

deploy:
  stage: staging
  <<: *mobile-tags
  variables:
    ANDROID_AAB_FILE: ${CI_PROJECT_DIR}/app-release.aab
    IOS_IPA_FILE: ${CI_PROJECT_DIR}/CustomerApp.ipa
  dependencies:
    - build
  <<: *mobile-before-script
  script:
    - fastlane android deploy
    - fastlane ios deploy
  when: manual
```
