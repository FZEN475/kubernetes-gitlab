# kubernetes-gitlab



























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
      <td>migrations</td>
      <td>При восстановлении базы с новыми секретами:<br/>OpenSSL::Cipher::CipherError</td>
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
  <tr>
      <td>Vault</td>
      <td>В логах аутентификации: <br/> permission denied<br/>"path":"auth/token/lookup-self"</td>
      <td>

У политики default нет разрешений на чтение пути.<br/>
```json
path "auth/token/lookup-self" {
    capabilities = ["read"]
}
```
</td>
  </tr>
  <tr>
      <td>prometheus-adapter</td>
      <td>Не устанавливаются APIService при установке через helm.</td>
      <td>

Заполнить:<br/>.Values.rules.custom<br/>.Values.rules.external<br/>.Values.rules.resource<br/>
Любыми правилами. Потом можно менять редактированием ConfigMap.
</td>
  </tr>
  <tr>
      <td>kube-rbac-proxy</td>
      <td>После успешного развёртывания нет доступа к бекэнду с корректным JWT-токеном.</td>
      <td>

В настройках ConfigMap очистить subresource.
</td>
  </tr>
</table>