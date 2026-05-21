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
├── register.yml                # Playbook principal
├── vars.yml                    # Variáveis de configuração
└── collections/
    └── requirements.yml        # Collections Ansible requeridas
```

## Requisitos

- ansible-core >= 2.14
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
| `vcenter_hostname` | Hostname ou IP do vCenter | Sim |
| `vcenter_username` | Usuário do vCenter | Sim |
| `vcenter_password` | Senha do vCenter | Sim |
| `vcenter_validate_certs` | Validar certificados SSL do vCenter | Sim |
| `vcenter_datacenter` | Nome do datacenter no vCenter | Sim |
| `vcenter_folder_path` | Nome da folder de VMs (ex: `Linux`) | Não |
| `vcenter_tag` | Nome da tag para filtrar VMs | Não |
| `satellite_url` | URL do Satellite (ex: `https://satellite.example.com`) | Sim |
| `satellite_username` | Usuário do Satellite | Sim |
| `satellite_password` | Senha do Satellite | Sim |
| `satellite_hostgroup` | Host group para registro | Sim |
| `vm_ansible_user` | Usuário SSH para acessar as VMs | Sim |

### Filtros opcionais

Os filtros `vcenter_folder_path` e `vcenter_tag` podem ser usados juntos, separados ou nenhum:

| `vcenter_folder_path` | `vcenter_tag` | Comportamento |
|---|---|---|
| Não definida | Não definida | Todas as VMs RHEL do vCenter |
| Definida | Não definida | VMs RHEL da folder |
| Não definida | Definida | Todas as VMs RHEL com a tag |
| Definida | Definida | VMs RHEL da folder com a tag |

Se a folder especificada não existir, o playbook encerra sem erro.

## Execução

### Local

```bash
ansible-playbook register.yml
```

### Red Hat Ansible Automation Platform

1. Configure o projeto apontando para este repositório
2. As collections serão instaladas automaticamente no sync
3. Crie um Job Template usando o playbook `register.yml`
4. Defina as variáveis no template ou via survey
5. Use o Execution Environment `ee-supported-rhel9`

## Segurança

O arquivo `vars.yml` contém credenciais em texto plano. Recomenda-se:

- Usar **Ansible Vault** para encriptar as senhas: `ansible-vault encrypt vars.yml`
- No AAP, usar **Credentials** para injetar as credenciais via extra-vars ou variáveis de ambiente
