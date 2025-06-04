# infra-kvm-hipervisor

KVM (Kernel-based Virtual Machine) é uma solução de virtualização de código aberto para sistemas operacionais Linux. Ela permite que você execute múltiplas máquinas virtuais (VMs) independentes, cada uma com seu próprio sistema operacional, em um único servidor físico. O KVM funciona como um módulo do kernel Linux e usa recursos de hardware, como CPU, memória e dispositivos de armazenamento, para criar ambientes isolados e eficientes para diferentes sistemas operacionais e aplicativos. É amplamente utilizado em ambientes de data center, testes e desenvolvimento devido à sua escalabilidade e desempenho.

## Requisitos

Antes de criar ou gerenciar suas máquinas virtuais, é importante verificar se seu hardware suporta virtualização. Os comandos abaixo ajudam nisso:

```bash
# Verifica se sua CPU suporta virtualização x86 (Intel VT-x ou AMD-V)
egrep -c '(vmx|svm)' /proc/cpuinfo

# Instala a ferramenta de verificação de CPU para virtualização
apt install cpu-checker

# Alternativa para verificar se o seu sistema suporta KVM
kvm-ok
```

## Instalação

Para configurar o ambiente de virtualização com KVM no seu sistema Ubuntu, execute os comandos a seguir para instalar os componentes necessários, configurar permissões e verificar o status:

```bash
# Instala os componentes essenciais do KVM e virt-manager, além de bridge-utils para configurar redes bridged
apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils -y

# Adiciona o usuário 'glauber' ao grupo 'libvirt' para gerenciar virtualizações
adduser glauber libvirt

# Adiciona o usuário 'glauber' ao grupo 'kvm' para acesso a recursos de hardware de virtualização
adduser glauber kvm

# Lista todas as VMs gerenciadas pelo libvirt
virsh list --all

# Habilita e inicia o serviço do libvirt para gerenciar as máquinas virtuais automaticamente na inicialização
systemctl enable --now libvirtd

# Instala a interface gráfica de gerenciamento de VM (Virtual Machine Manager)
apt install virt-manager -y

# Reconfirma a lista de VMs
virsh list --all

```


## Configuração da rede

## Criando VMs

## Comandos e conexão

## Migração

## Endereçamento IP

## Backups

## Referencias