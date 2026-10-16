# ansible-examples

Exemplos de playbooks prontos para serem carregados como projeto no
**Ansible Automation Platform (AAP)**.

## Estrutura

```
.
├── collections/
│   └── requirements.yml      # coleções instaladas pelo AAP na sincronização
├── playbooks/
│   └── openshift/
│       └── create_project.yml
└── aap/
    └── surveys/              # surveys de exemplo para os Job Templates
        └── openshift_create_project.json
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

## Playbooks

### `playbooks/openshift/create_project.yml`

Cria um projeto vazio no OpenShift, equivalente a `oc new-project`. Se o
projeto já existir, nada é alterado.

| Variável               | Obrigatória | Descrição                              |
|------------------------|-------------|----------------------------------------|
| `project_name`         | sim         | Nome do projeto (padrão DNS, até 63 caracteres) |
| `project_display_name` | não         | Nome de exibição no console            |
| `project_description`  | não         | Descrição do projeto                   |

**Credencial:** tipo *OpenShift or Kubernetes API Bearer Token*, com a URL da
API do cluster (ex.: `https://api.cluster.exemplo.com:6443`) e o token de um
usuário ou service account com permissão para criar projetos
(`self-provisioner` ou superior).

**Execution Environment:** precisa da coleção `kubernetes.core` e da biblioteca
Python `kubernetes`. O EE suportado padrão do AAP (`ee-supported-rhel9`) já
atende.

Execução local para teste:

```bash
ansible-galaxy collection install -r collections/requirements.yml
export K8S_AUTH_HOST=https://api.cluster.exemplo.com:6443
export K8S_AUTH_API_KEY=$(oc whoami -t)
ansible-playbook playbooks/openshift/create_project.yml -e project_name=meu-projeto
```
