# Ansible Satellite Register VMware VMs

Playbook Ansible que conecta ao VMware vCenter, descobre VMs e as registra automaticamente no Red Hat Satellite.

## Como funciona

1. Conecta ao vCenter e lista as VMs (com filtros opcionais por folder e tag)
2. Filtra apenas VMs Red Hat Enterprise Linux com IP reportado pelo VMware Tools
3. Cria um inventário dinâmico em memória com as VMs descobertas
4. Remove o registro atual do subscription-manager (se houver)
5. Gera o comando de registro via API do Satellite e o executa em cada VM

## Estrutura

```
├── register.yml                    # Playbook principal
├── vars.yml                        # Variáveis de configuração
├── collections/
│   └── requirements.yml            # Collections Ansible requeridas
└── ee/
    └── execution-environment.yml   # Definição do Execution Environment
```

## Requisitos

- ansible-core >= 2.16
- Acesso ao vCenter com permissões de leitura
- Acesso ao Red Hat Satellite com permissões de registro
- VMware Tools instalado e rodando nas VMs alvo
- Acesso SSH às VMs via usuário configurado em `vm_ansible_user`

### Collections

As collections são instaladas automaticamente pelo AAP durante o sync do projeto. Para instalação manual:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Configuração

Edite o arquivo `vars.yml` com os dados do seu ambiente:

| Variável | Descrição | Obrigatória |
|---|---|---|
| `vcenter_hostname` | Hostname ou IP do vCenter | Sim* |
| `vcenter_username` | Usuário do vCenter | Sim* |
| `vcenter_password` | Senha do vCenter | Sim* |
| `vcenter_validate_certs` | Validar certificados SSL do vCenter | Não |
| `vcenter_datacenter` | Nome do datacenter no vCenter | Sim |
| `vcenter_folder_path` | Nome da folder de VMs (ex: `Linux`) | Não |
| `vcenter_tag` | Nome da tag para filtrar VMs | Não |
| `satellite_url` | URL do Satellite (ex: `https://satellite.example.com`) | Sim |
| `satellite_username` | Usuário do Satellite | Sim |
| `satellite_password` | Senha do Satellite | Sim |
| `satellite_hostgroup` | Host group para registro | Sim |
| `vm_ansible_user` | Usuário SSH para acessar as VMs | Sim |

\* As credenciais do vCenter podem ser fornecidas via `vars.yml` ou via credential do tipo **VMware vCenter** no AAP. Quando a credential está configurada no AAP, as variáveis de ambiente `VMWARE_HOST`, `VMWARE_USER` e `VMWARE_PASSWORD` são injetadas automaticamente e usadas como fallback.

### Filtros opcionais

Os filtros `vcenter_folder_path` e `vcenter_tag` podem ser usados juntos, separados ou nenhum:

| `vcenter_folder_path` | `vcenter_tag` | Comportamento |
|---|---|---|
| Não definida | Não definida | Todas as VMs RHEL do vCenter |
| Definida | Não definida | VMs RHEL da folder |
| Não definida | Definida | Todas as VMs RHEL com a tag |
| Definida | Definida | VMs RHEL da folder com a tag |

Se a folder especificada não existir, o playbook encerra sem erro.

### Tags

O playbook usa tags para permitir execução parcial:

| Comando | Comportamento |
|---|---|
| `ansible-playbook register.yml` | Executa tudo (descoberta + registro) |
| `ansible-playbook register.yml --skip-tags register` | Apenas descoberta de VMs |

No AAP, configure **Skip Tags: `register`** no Job Template para rodar apenas a descoberta.

## Execution Environment

O EE padrão do AAP não inclui a biblioteca `pyVmomi`, necessária para os módulos `community.vmware`. É preciso construir um EE customizado.

### Pré-requisitos

- `ansible-builder` instalado (`pip install ansible-builder`)
- `podman` instalado
- Acesso ao registry `registry.redhat.io` (`podman login registry.redhat.io`)

### Build do EE

```bash
cd ee
ansible-builder build -t acarlos-pyvmomi:latest -f execution-environment.yml -v3
```

A imagem é baseada em `ee-minimal-rhel9` (AAP 2.5, ansible-core 2.16) com `pyVmomi` adicionado.

### Push para o Private Automation Hub

```bash
podman tag acarlos-pyvmomi:latest <registry-do-aap>/acarlos-pyvmomi:latest
podman login <registry-do-aap>
podman push <registry-do-aap>/acarlos-pyvmomi:latest
```

Substitua `<registry-do-aap>` pela URL do registry do seu Private Automation Hub.

### Registro no AAP

Após o push, o EE aparece em **Automation Content > Execution Environments**. Caso não apareça, crie manualmente:

1. Vá em **Automation Content > Execution Environments**
2. Clique em **Create execution environment**
3. Preencha **Name** e **Image** com `<registry-do-aap>/acarlos-pyvmomi:latest`

## Execução

### Local

```bash
ansible-playbook register.yml
```

### Red Hat Ansible Automation Platform

1. Configure o projeto apontando para este repositório
2. Construa e publique o EE customizado (ver seção acima)
3. Configure as **Galaxy Credentials** na organização:
   - **1o**: Automation Hub (para `redhat.satellite`)
   - **2o**: Ansible Galaxy (para `community.vmware`)
4. As collections serão instaladas automaticamente no sync do projeto
5. Crie um Job Template usando o playbook `register.yml`
6. Selecione o EE `acarlos-pyvmomi` no template
7. Defina as variáveis no template ou via survey

## Segurança

O arquivo `vars.yml` contém credenciais em texto plano. Recomenda-se:

- Usar **Ansible Vault** para encriptar as senhas: `ansible-vault encrypt vars.yml`
- No AAP, usar **Credentials** para injetar as credenciais via extra-vars ou variáveis de ambiente
