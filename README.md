# Kubernetes секреты из Vault используя external-secrets-operator

## Введение
В этой статье будет описано как создать kubernetes секреты из Vault с помощью AppRole используя 
external-secrets-operator. Также будет использован Terragrunt.

## Kubernetes кластер
Для начала создадим Kubernetes кластер (например, в Яндекс облаке с помощью Terragrunt).

Создание kubernetes кластера примерно описана в файле [create-k8s-by-terraform-in-yc.md](terragrunt-k8s/create-k8s-by-terraform-in-yc.md)

# Установка hashicorp vault. Если у вас hashicorp vault, то пропускаем раздел.
```shell
helm install vault oci://registry-1.docker.io/bitnamicharts/vault --version 0.2.1 -n vault --create-namespace
```

- Инициализация и распечатывание хранилища. Ждем когда vault-server-0 перейдет в состояние Running.
```shell
$ kubectl get pods --namespace "vault" -l app.kubernetes.io/instance=vault
NAME                              READY   STATUS    RESTARTS   AGE
vault-injector-6f8fb5dcff-bbl2s   0/1     Running   0          14s
vault-server-0                    0/1     Pending   0          7s
```

- Инициализируйте один сервер хранилища с количеством общих ключей по умолчанию и пороговым значением ключа по умолчанию:
```shell
$ kubectl exec -ti vault-server-0 -n vault -- vault operator init
Unseal Key 1: 99eImNoiWWC+SLfi8IV82D9Kgqkyg/xUUKHQGHyutMlC
Unseal Key 2: 99wTfMKGZ7qE1vcdh8EH599fGaSqkWMtp3+a7VCxbY3k
Unseal Key 3: VF5Ilga1PgPMWJxuB+sNlT0mOHi/GlHGjQQO3IuZ8Dzi
Unseal Key 4: vA7UchKrBGisiygUG9o/P+Y+7yx5s6pWMyM/D+D4F5rp
Unseal Key 5: k92YdBmIXAH3m9rP84gcsXAtgNhHcZqD43KP4D97+e3N

Initial Root Token: hvs.EsdGfUGUElenNaiThSDP7vlY
```

- В выходных данных отображаются общие ключи и сгенерированный исходный корневой ключ. Распечатайте сервер hashicorp vault с общими ключами до тех пор, пока не будет достигнуто пороговое значение ключа:
```shell
$ kubectl exec -ti vault-server-0 -n vault -- vault operator unseal # ... Unseal Key 1
$ kubectl exec -ti vault-server-0 -n vault -- vault operator unseal # ... Unseal Key 2
$ kubectl exec -ti vault-server-0 -n vault -- vault operator unseal # ... Unseal Key 3
```

- В отдельном терминале пробросьте порт 8200 от hashicorp vault
```shell 
kubectl port-forward vault-server-0 8200:8200 -n vault
```

- Экспортируйте адрес hashicorp vault
```shell
export VAULT_ADDR=http://127.0.0.1:8200
```

Вы можете либо настроить AppRole в Vault из CLI либо через terraform код.

Настроим AppRole Vault через terraform.
Переходим в директорию с файлом `setting-approle-vault.tf`
```shell
cd vault-resource
```

Экспортируем ваш токен (в данном случае root токен)
```shell
export VAULT_TOKEN=hvs.EsdGfUGUElenNaiThSDP7vlY
```

Применим конфигурацию terraform.
```shell
terraform init
terraform apply
```

- Создайте секрет из CLI.
```shell
vault kv put data/postgres POSTGRES_USER=admin POSTGRES_PASSWORD=123456
```

Либо создайте Vault секрет через UI как показано на скриншоте:
![Create-vault-secret-from-cli.png](vault-resource/Create-vault-secret-from-cli.png)

Выведите на экран терминала role-id и secret-id.
B выходим из директории vault-resource.
```shell
terraform output role_id
terraform output secret_id
cd ..
```

Если вам интересно настроить AppRole в Vault из CLI, то настройка описано в отдельном файле 
[create-approle-vault-cli.md](vault-resource/create-approle-vault-cli.md)


Добавляем helm репо External Secrets Operator
```shell
helm repo add external-secrets https://charts.external-secrets.io
```

Устанавливаем External Secrets Operator
```shell
helm install external-secrets \
external-secrets/external-secrets \
    --wait \
    -n external-secrets \
    --create-namespace \
    --version 0.8.3 \
    --set installCRDs=true
```

Настройка external-secrets.
 - Указываем `role_id` в файле `external-secrets/secret-store.yaml` в поле `roleId`.
 - Указываем `secret_id` в файле `external-secrets/vault-secret.yaml` в поле `secret-id`.
Файлы конфигурации в директории external-secrets подробно документированы.


Применяем yaml из директории external-secrets
```shell
kubectl apply -f external-secrets
```

Дебаг:
ClusterSecretStore c названием vault-backend должен иметь CAPABILITIES - ReadWrite.
```shell
$ kubectl get ClusterSecretStore vault-backend
NAME            AGE     STATUS   CAPABILITIES   READY
vault-backend   3m19s   Valid    ReadWrite      True
```

Если ExternalSecret имеет статус SecretSyncedError, то смотрим describe.
```shell
$ kubectl get ExternalSecret -n external-secrets external-secret
NAMESPACE          NAME              STORE           REFRESH INTERVAL   STATUS              READY
external-secrets   external-secret   vault-backend   5s                 SecretSyncedError   False
```

# Links:
 - https://github.com/fvoges/terraform-vault-basic-workflow
 - https://github.com/tiwarisanjay/external-secrets-operator
 - https://github.com/tiwarisanjay/argocd-everything/blob/main/argocd-ha-vault-sso/README.md
 - https://artifacthub.io/packages/helm/bitnami/vault
 - https://gengwg.medium.com/setting-up-flux-v2-with-kind-cluster-and-github-on-your-laptop-56e28b0a8120
 - https://github.com/hashicorp/terraform-provider-vault/blob/main/website/docs/r/mount.html.md
 - https://earthly.dev/blog/eso-with-hashicorp-vault/
