# kubernetes-gitlab
## Description
* Установка через helm чарт.
* Для удобства [values.yaml](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_1_global.yaml) разделён по сервисам.
### Используемые [компоненты gitlab](https://docs.gitlab.com/development/architecture/)
* [migration](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_2_migration.yaml) - инициализация и обновление баз данных.
* [webservice](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_3_webservice.yaml) - сервер приложений Puma, запускающий Rails. Web интерфейс и обработка API запросов. 
* [workhorse](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_3_webservice.yaml) - обработка длительных запросов к web-сервису.
* [sidekiq](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_4_sidekiq.yaml) - обработка заданий gitlab в фоновом режиме. (Например раннеры).
* [gitaly](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_5_gitaly.yaml) - Обработка вызовов git. Хранение файлов проекта.
  * Файлы проектов хранятся прямо в контейнере. (Медленный NAS).
  * Для резервного копирования используется дополнительный контейнер с cron.
  * Для восстановления используется init контейнер.
* [shell](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_6_shell.yaml) - управление доступом к git по ssh.
* [registry](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_7_registry.yaml) - хранение изображений docker.
* [kas](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_8_kas.yaml) - интеграция с kubernetes. Агент мониторинга кластера kubernetes поддерживает актуальное состояние объектов в соответствии с проектом git.
* [exporter](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_9_exporter.yaml) - предоставление метрик gitlab.
* [pages](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_1_gitlab/_10_pages.yaml) - публикация статический web-сайтов из репозитория.
* runners - выполнение заданий CI/CD.
  * Токены регистрации создаются в gitlab и обновляются в Vault.
  * <details><summary> Runners list. </summary>

    | Name                                                                                                              | Default image                                         | Comment                   |
    |:------------------------------------------------------------------------------------------------------------------|:------------------------------------------------------|:--------------------------|
    | [no_tag](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_2_runners/_1_no_tag.yaml)                 | [multitool](https://github.com/FZEN475/multitool.git) | Выполнение общих заданий. |
    | [docker_builder](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_2_runners/_2_docker_builder.yaml) | [kaniko](gcr.io/kaniko-project/executor:debug)        | Сборка образов.           |
    | [helm](https://github.com/FZEN475/kubernetes-gitlab/blob/main/config/_2_runners/_4_helm.yaml)     | [alpine/helm](https://hub.docker.com/r/alpine/helm)           | Доступ к kubernetes       |
  
  </details>
* agents - доступ к kubernetes.
  * Токены регистрации создаются в gitlab и обновляются в Vault.
  * Агенты используют сервисные аккаунты без доступа ко всему кластеру.
  * Список агентов и их настройки в [репозитории](https://github.com/FZEN475/kubernetes-agent-config.git).
  * <details><summary> Agents list. </summary>

    | Name                  | Namespace | Comment                                 |
    |:----------------------|:----------|:----------------------------------------|
    | kubernetes-agent-dev  | dev       | Имеет полные права в пространстве имён. |
    | kubernetes-agent-prod | prod       | Имеет полные права в пространстве имён. |

  </details>
  
## Dependency

* [Образ](https://github.com/FZEN475/ansible-image)
* [Library](https://github.com/FZEN475/ansible-library)
* <details><summary> .env </summary>

  ```properties
  TERRAFORM_REPO="https://github.com/FZEN475/kubernetes-gitlab.git"
  #GIT_EXTRA_PARAM="-btemp_branch"
  SECURE_SERVER=""
  SECURE_PATH=""
  LIBRARY="https://github.com/FZEN475/ansible-library.git"
  ``` 
  </details>

* <details><summary> secrets </summary>

  ```yaml
  secrets:
    - id_ed25519
  ```
</details>


## Stages

### [init](https://github.com/FZEN475/kubernetes-gitlab/blob/main/playbooks/_0_init/_1_install.yaml)
* Создание PVC для резервных копий.
* Создание ConfigMap со скриптами резервного копирования и восстановления. Используются в дополнительных контейнерах gitaly.
### [gitlab](https://github.com/FZEN475/kubernetes-gitlab/blob/main/playbooks/_1_gitlab/_1_install.yaml)
* Создание общего ServiceAccount для gitlab и доступа к секретам gitlab в vault.
* Установка gitlab.
* [Fix: gitlab-registry, gitlab-sidekiq-all-in-1-v2, gitlab-toolbox, gitlab-webservice-default](https://github.com/FZEN475/kubernetes-gitlab?tab=readme-ov-file#Troubleshoots)
### [runners](https://github.com/FZEN475/kubernetes-gitlab/blob/main/playbooks/_2_runners/_1_install.yaml)
* Создание и последующее удаление DaemonSet для копирования актуальных ca.crt registry на каждую NODE.
* Создание PVC для кеширования на NFS.
* Установка runners.
### [agents](https://github.com/FZEN475/kubernetes-gitlab/blob/main/playbooks/_3_agents/_1_install.yaml)
* Создание ServiceAccount с необходимыми правами для агентов.
* Установка агентов.

## Troubleshoots

<!DOCTYPE html>
<table>
  <thead>
    <tr>
      <th>Источник</th>
      <th>Проблема</th>
      <th>Решение</th>
    </tr>
  </thead>
  <tr>
      <td>gitlab</td>
      <td>В чарте gitlab название сертификата registry-auth.crt жестко прописано и не редактируется через values.yaml<br/>Это не позволяет управлять сертификатами cert-manager.</td>
      <td>

Необходимо ручное редактирование конфигурации gitlab.
</td>
  </tr>
  <tr>
      <td>migrations</td>
      <td>При восстановлении базы с новыми секретами:<br/>OpenSSL::Cipher::CipherError<br/><br/><br/>Но в моём случае восстановить базу без секретов не удалось.</td>
      <td>

Старые зашифрованные секреты невозможно прочитать. Нужно очистить нечитаемые секреты.<br/>
```shell
kubectl exec -it -n gitlab             pod/gitlab-toolbox-xxxxxxxxx-xxxxx -- bash
gitlab-rails console
settings = ApplicationSetting.last
settings.update_column(:runners_registration_token_encrypted, nil)
exit
# Если не помогло.
# Список нечитаемых секретов
gitlab-rake gitlab:doctor:secrets VERBOSE=1
# Некоторые токены не очистятся без изменения базы данных. (OpenSSL::Cipher::CipherError:)
# ApplicationSetting: ci_jwt_signing_key, runners_registration_token, error_tracking_access_token...
# Очистить нужно сразу все.
# [Ссылка](https://forum.gitlab.com/t/web-ide-500/117193/22)
kubectl exec -it -n storage            pod/postgresql-primary-0 -- bash
psql -U postgres
use gitlabhq_production;
SELECT error_tracking_access_token_encrypted from application_settings;
UPDATE application_settings SET error_tracking_access_token_encrypted = null;
# Очистка секретов value в разделе Ci::PipelineVariable (DRY_RUN=false - применяет изменения)
gitlab-rake gitlab:doctor:reset_encrypted_tokens MODEL_NAMES=Ci::PipelineVariable TOKEN_NAMES=value VERBOSE=true DRY_RUN=false
```
</td>
  </tr>
</table>