# ansible-examples

Exemplos de playbooks prontos para serem carregados como projeto no
**Ansible Automation Platform (AAP)**.

## Estrutura

```
.
├── collections/
│   └── requirements.yml          # coleções instaladas pelo AAP na sincronização
├── playbooks/
│   ├── timer.yml                 # Timer (espera de 5 min, para testes de timeout)
│   └── openshift/
│       ├── create_project.yml    # CreateProject
│       ├── deploy_httpd.yml      # DeployHttpd
│       └── tasks/
│           └── check_credentials.yml   # tarefas compartilhadas
└── aap/
    └── surveys/                  # surveys de exemplo para os Job Templates
        ├── openshift_create_project.json
        └── openshift_deploy_httpd.json
```

## Configurando no AAP

1. **Projeto** – crie um Project com *Source Control Type* = Git apontando para
   `https://github.com/victorfim/ansible-examples.git`. Na sincronização o AAP
   instala as coleções de `collections/requirements.yml`.
2. **Credencial** – crie as credenciais exigidas pelo playbook (veja abaixo).
3. **Job Template** – escolha o projeto, o playbook, um inventário (pode ser um
   inventário vazio; os playbooks rodam em `localhost`) e a credencial.
4. **Survey** (opcional) – use o JSON correspondente em `aap/surveys/` como
   referência para cadastrar as perguntas.

### Credencial e Execution Environment (OpenShift)

**Credencial:** tipo *OpenShift or Kubernetes API Bearer Token*, com a URL da
API do cluster (ex.: `https://api.cluster.exemplo.com:6443`) e o token de um
usuário ou service account com permissão para criar projetos
(`self-provisioner` ou superior). A mesma credencial serve para os dois
playbooks.

**Execution Environment:** precisa da coleção `kubernetes.core` e da biblioteca
Python `kubernetes`. O EE suportado padrão do AAP (`ee-supported-rhel9`) já
atende.

## Playbooks

### CreateProject – `playbooks/openshift/create_project.yml`

Cria um projeto vazio no OpenShift (equivalente a `oc new-project`). Se o
projeto já existir, nada é alterado.

| Variável               | Obrigatória | Padrão | Descrição                              |
|------------------------|-------------|--------|----------------------------------------|
| `project_name`         | sim         |        | Nome do projeto (padrão DNS, até 63 caracteres) |
| `project_display_name` | não         | `""`   | Nome de exibição no console            |
| `project_description`  | não         | `""`   | Descrição do projeto                   |

Publica `project_name` com `set_stats`.

### DeployHttpd – `playbooks/openshift/deploy_httpd.yml`

Implanta uma aplicação httpd de exemplo num projeto **que já existe**:

- **Deployment** com a imagem `httpd` do registry interno do cluster
  (`image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest`)
- **Service** na porta 8080
- **Route** HTTPS (TLS edge, com redirecionamento de HTTP para HTTPS)

O playbook espera o Deployment ficar disponível e mostra a URL no final. Tudo
é idempotente: rodar de novo não duplica nada.

| Variável       | Obrigatória | Padrão  | Descrição                              |
|----------------|-------------|---------|----------------------------------------|
| `project_name` | sim         |         | Projeto de destino (precisa existir)   |
| `app_name`     | não         | `httpd` | Nome do Deployment, Service e Route    |
| `app_image`    | não         | imagem `httpd` do registry interno | Imagem do container |

Publica `project_name` e `app_url` com `set_stats`.

A imagem padrão vem da ImageStream `httpd` do namespace `openshift`, criada
pelo Samples Operator. Se ela não existir no seu cluster, informe outra imagem
em `app_image`.

### Encadeando os dois num Workflow

Crie um Workflow Template com o nó **CreateProject** seguido de
**DeployHttpd** (na saída *On Success*). Como o CreateProject publica
`project_name` com `set_stats`, o DeployHttpd recebe o valor automaticamente.
Coloque o survey no Workflow (com `project_name` e, se quiser, `app_name`) em
vez de nos Job Templates individuais.

### Timer – `playbooks/timer.yml`

Aguarda 5 minutos e termina com uma mensagem de sucesso. Serve para testar
timeouts de outras aplicações (por exemplo, o timeout de um Job Template do
AAP). Roda em `localhost`, não precisa de credencial nem de survey.

| Variável        | Obrigatória | Padrão | Descrição                    |
|-----------------|-------------|--------|------------------------------|
| `timer_seconds` | não         | `300`  | Duração da espera em segundos |

## Execução local para teste

```bash
ansible-galaxy collection install -r collections/requirements.yml
export K8S_AUTH_HOST=https://api.cluster.exemplo.com:6443
export K8S_AUTH_API_KEY=$(oc whoami -t)
ansible-playbook playbooks/openshift/create_project.yml -e project_name=meu-projeto
ansible-playbook playbooks/openshift/deploy_httpd.yml   -e project_name=meu-projeto
ansible-playbook playbooks/timer.yml                       # ou: -e timer_seconds=60
```
