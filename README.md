# Cluster Kubernetes com K3s e Ansible

Automação de um homelab com um servidor K3s e um worker, usando máquinas Debian,
acesso SSH pela rede de gerenciamento e uma interface Ethernet dedicada à rede
privada do cluster.

## Topologia

Configuração atual em `ansible/inventory/hosts.yml`:

| Host | Papel | IP de gerenciamento (SSH) | IP privado do cluster |
| --- | --- | --- | --- |
| `k8s-node01` | Servidor K3s | `192.168.0.200` | `10.10.10.11` |
| `k8s-node02` | Worker K3s | `192.168.0.201` | `10.10.10.12` |

A interface privada é `enp2s0f1`, configurada com prefixo `/24`, sem gateway.
O worker conecta-se ao servidor em `https://10.10.10.11:6443`.
Esta configuração tem um único servidor; não implementa alta disponibilidade.

## Estrutura

```text
ansible/
├── inventory/hosts.yml
└── playbooks/
    ├── bootstrap.yml                 # Pacotes básicos, SSH e /etc/hosts
    ├── configure-cluster-network.yml # Endereçamento privado via ifupdown
    ├── prepare-k3s.yml               # Diretórios, módulos, sysctl e swap
    ├── install-k3s-server.yml        # Configuração e instalação do servidor
    └── install-k3s-worker.yml        # Leitura do token e instalação dos workers
```

## Pré-requisitos

- No controlador: Ansible, cliente SSH e acesso aos IPs de gerenciamento.
- Nos nós: Debian com APT, Python 3 em `/usr/bin/python3`, SSH e usuário `debian`
  com permissão de elevação via sudo.
- Acesso dos nós à internet para os pacotes e o instalador `https://get.k3s.io`.
- Interface privada conectada entre os nós e endereços livres na rede `10.10.10.0/24`.
- O arquivo `/etc/network/interfaces` deve incluir `/etc/network/interfaces.d/*`.
  A interface privada deve estar sob controle do ifupdown, sem configuração
  concorrente por outro gerenciador de rede.

## Configuração

Edite o inventário antes de executar:

| Variável | Valor atual | Uso |
| --- | --- | --- |
| `ansible_user` | `debian` | Usuário SSH |
| `ansible_python_interpreter` | `/usr/bin/python3` | Python remoto |
| `ansible_host` | Por host | Endereço de gerenciamento |
| `cluster_ip` | Por host | Endereço privado e IP do nó K3s |
| `k3s_flannel_iface` | `enp2s0f1` | Interface privada usada pelo Flannel |
| `k3s_data_dir` | `/home/k3s-data` | Dados internos do K3s |
| `k3s_storage_dir` | `/home/k3s-storage` | Caminho configurado para armazenamento local |

O bloco de nomes privados em `bootstrap.yml` contém IPs e nomes fixos: atualize-o
também se mudar a topologia. Mantenha apenas um host em `k3s_server`; o playbook
dos workers usa o primeiro servidor desse grupo.

## Execução

Execute os comandos na raiz deste repositório. Acrescente `--ask-become-pass`
quando o sudo exigir senha e `--private-key /caminho/da/chave` se necessário.

Confira inventário e conectividade:

```bash
ansible-inventory -i ansible/inventory/hosts.yml --graph
ansible all -i ansible/inventory/hosts.yml -m ansible.builtin.ping
```

Execute os playbooks nesta ordem:

```bash
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/bootstrap.yml
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/configure-cluster-network.yml
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/prepare-k3s.yml
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/install-k3s-server.yml
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/install-k3s-worker.yml
```

A configuração de rede pode reiniciar a interface privada. A preparação desativa
swap e comenta suas entradas em `/etc/fstab`, além de configurar módulos e
encaminhamento IP. As instalações escrevem `/etc/rancher/k3s/config.yaml` e
habilitam os serviços `k3s` e `k3s-agent`.

O playbook de workers primeiro lê o token no servidor e o mantém em memória no
Ansible, com `no_log`. Por isso, não execute esse playbook com um `--limit` que
exclua o servidor. A configuração do agent contém o token e recebe modo `0600`.

## Verificação e acesso

Após a instalação, confira os nós e os pods pelo servidor:

```bash
ssh debian@192.168.0.200 'sudo k3s kubectl get nodes -o wide'
ssh debian@192.168.0.200 'sudo k3s kubectl get pods -A'
```

Para usar um cliente `kubectl` local, copie o kubeconfig com permissões restritas:

```bash
(umask 077; ssh debian@192.168.0.200 'sudo cat /etc/rancher/k3s/k3s.yaml' > k3s.yaml)
```

Esse comando pressupõe sudo sem prompt remoto. No arquivo local, substitua o
endereço do campo `server` por `https://192.168.0.200:6443`, acessível pelo
controlador e incluído nos SANs configurados pelo playbook. Depois execute:

```bash
kubectl --kubeconfig ./k3s.yaml get nodes -o wide
```

O kubeconfig concede acesso administrativo: não o versione. O `.gitignore`
exclui `k3s.yaml` e mantém a exclusão preexistente de `ks3.yaml`.

## Validação local

Valide a sintaxe sem aplicar alterações remotas:

```bash
for playbook in ansible/playbooks/*.yml; do
  ansible-playbook -i ansible/inventory/hosts.yml "$playbook" --syntax-check || break
done
```

Essa checagem não comprova conectividade, resolução de todas as variáveis em
tempo de execução ou funcionamento do cluster.

## Operação e limitações

- Mudanças na configuração K3s notificam handlers que reiniciam o serviço.
- O instalador é executado somente quando `/usr/local/bin/k3s` não existe.
  Reexecutar os playbooks não implementa atualização de versão ou troca de papel.
- A versão do K3s não está fixada e o instalador é baixado da URL oficial.
- Os playbooks não configuram firewall, backups, restauração ou armazenamento
  distribuído. O diretório local de storage não fornece replicação entre nós.
- A rede aplica o IP desejado, mas não remove endereços antigos; mudanças de IP
  exigem conferir a configuração ativa da interface.

Para investigar falhas nos serviços:

```bash
ssh debian@192.168.0.200 'sudo journalctl -u k3s -n 100 --no-pager'
ssh debian@192.168.0.201 'sudo journalctl -u k3s-agent -n 100 --no-pager'
```

Se um worker não ingressar, confira a interface privada, a comunicação com o IP
do servidor e se o servidor foi instalado antes da execução dos workers.
