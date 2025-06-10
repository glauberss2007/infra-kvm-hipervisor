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

### 1. `qemu-kvm`
- **Propósito:** Fornece o hipervisor KVM (Kernel-based Virtual Machine), permitindo que seu sistema execute máquinas virtuais de forma eficiente.
- **Detalhes:** O QEMU é uma ferramenta de emulação que também oferece funcionalidades de virtualização com aceleração de hardware através do KVM. Ao instalar o `qemu-kvm`, seu sistema consegue criar e rodar máquinas virtuais usando hardware de virtualização (como CPU Intel VT-x ou AMD-V).

### 2. `libvirt-daemon-system`
- **Propósito:** É o daemon principal do libvirt responsável por gerenciar a virtualização.
- **Detalhes:** Este pacote hospeda o serviço `libvirtd`, que gerencia a criação, monitoramento e destruição de máquinas virtuais e recursos de rede/storages, atuando como uma interface central para diferentes tecnologias de virtualização (QEMU, KVM, Xen, etc.).

### 3. `libvirt-clients`
- **Propósito:** Contém as ferramentas de linha de comando para interagir com o libvirt.
- **Detalhes:** Fornece utilitários como o `virsh`, que permitem gerenciar virtualizações, redes, pools de armazenamento e outras funções de virtualização via linha de comando.

### 4. `bridge-utils`
- **Propósito:** Conjunto de utilitários para criar e gerenciar pontes de rede (network bridges).
- **Detalhes:** Bridges (pontes de rede) conectam as máquinas virtuais à rede externa de forma que elas possam receber endereçamento IP e se comportar como se estivessem fisicamente conectadas à rede. Com esse pacote, você pode configurar redes bridged para suas VMs.

## Configuração da rede

**1. Instalar o `bridge-utils`:**

```bash
sudo apt update
sudo apt install bridge-utils
```

**2. Identificar Sua Interface de Rede:**

Use `ip addr` para encontrar o nome da sua interface de rede física (ex: `eth0`, `enp0s3`, `wlan0`). Anote este nome.

**3. Encontrar o Arquivo de Configuração do Netplan:**

*   Liste os arquivos em `/etc/netplan/`:

    ```bash
    ls /etc/netplan/
    ```

    Você provavelmente verá um arquivo com um nome como `01-network-manager-all.yaml` ou `50-cloud-init.yaml`. O nome do arquivo pode variar. **Importante:** Identifique o arquivo YAML que parece estar configurando sua rede ativamente. Se você tiver vários arquivos, aquele com o prefixo numérico mais alto provavelmente é o ativo.

**4. Editar o Arquivo de Configuração do Netplan:**

*   Abra o arquivo YAML com privilégios de administrador usando `nano`:

    ```bash
    sudo nano /etc/netplan/SEU_ARQUIVO_NETPLAN.yaml  # Substitua SEU_ARQUIVO_NETPLAN.yaml pelo nome real do arquivo
    ```

*   Modifique o arquivo para criar a bridge. Aqui está um exemplo. **Importante:** Adapte isso ao seu arquivo existente, *preservando cuidadosamente* quaisquer configurações existentes (como `renderer: NetworkManager`, se estiver presente), a menos que você tenha certeza absoluta de que não precisa delas.

    ```yaml
    network:
      version: 2
      renderer: networkd  # Ou NetworkManager se o seu arquivo já o usa
      ethernets:
        eth0:  # Substitua 'eth0' pelo nome da sua interface física
          dhcp4: no
          dhcp6: no
          optional: true  # Impede falhas de inicialização se a interface não estiver presente

      bridges:
        br0:
          interfaces: [eth0]  # Substitua 'eth0' pelo nome da sua interface física
          dhcp4: no          # Desativa o DHCP na bridge
          dhcp6: no
          addresses: [192.168.1.100/24]  # Define um IP estático para a bridge
          gateway4: 192.168.1.1        # O IP do seu gateway/roteador
          nameservers:
              addresses: [8.8.8.8, 8.8.4.4]  # DNS do Google
    ```

    **Explicação:**

    *   `network: version: 2`: Especifica a versão da configuração do Netplan.
    *   `renderer: networkd` (ou `NetworkManager`): Isso é muito importante. O Ubuntu pode usar `networkd` ou `NetworkManager` para gerenciar a rede. **Se o seu arquivo original tinha `renderer: NetworkManager`, MANTENHA-O!** Se não estiver presente, `networkd` é o padrão. Usar o renderer errado pode causar problemas de rede.
    *   `ethernets:`: Configura interfaces Ethernet físicas.
        *   `eth0:` (Substitua pelo nome da sua interface real): Esta seção configura sua interface física.
            *   `dhcp4: no`, `dhcp6: no`: Desativa o DHCP na interface física. Ele receberá seu endereço IP através da bridge.
            *   `optional: true`: Impede falhas de inicialização se a interface não estiver presente. Boa prática.
    *   `bridges:`: Configura interfaces de bridge.
        *   `br0:`: Esta seção define sua interface de bridge.
            *   `interfaces: [eth0]` (Substitua pelo nome da sua interface real): Especifica que `eth0` é um membro da bridge.
            *   `dhcp4: no`, `dhcp6: no`: Desativa o DHCP na bridge (usaremos um IP estático).
            *   `addresses: [192.168.1.100/24]`: Define o endereço IP estático para a bridge (substitua por um IP livre em sua rede). `/24` é equivalente a uma máscara de rede de `255.255.255.0`.
            *   `gateway4: 192.168.1.1`: O IP do seu gateway/roteador.
            *   `nameservers: addresses: [8.8.8.8, 8.8.4.4]`: Servidores DNS.

**5. Aplicar a Configuração do Netplan:**

*   Execute o seguinte comando para aplicar as alterações:

    ```bash
    sudo netplan apply
    ```

    Se você vir erros, verifique cuidadosamente o arquivo YAML em busca de erros de sintaxe (a indentação é *muito* importante em YAML). Use `netplan try` para testar a configuração antes de aplicá-la permanentemente.

**6. Verificar:**

*   Use `ip addr show br0` para verificar se a interface `br0` tem o endereço IP correto e se sua interface física (ex: `eth0`) *não* tem um endereço IP.
*   Teste a conectividade de rede a partir da máquina host (ping seu gateway, ping um site público).

**7. Configurar o KVM:**

Ao criar suas máquinas virtuais, defina a interface de rede para usar "Bridge device: br0".

**Considerações Importantes:**

*   **Sintaxe YAML:** YAML é muito sensível à indentação. Certifique-se de que sua indentação está correta. Use espaços, não tabulações. A indentação deve ser consistente em todo o arquivo.
*   **Renderer:** Como mencionado acima, certifique-se de que está usando o renderer correto (`networkd` ou `NetworkManager`). Usar o errado causará problemas de rede. Se o seu arquivo YAML original tinha `renderer: NetworkManager`, você *deve* manter essa linha em seu arquivo modificado.
*   **Backup:** Antes de fazer qualquer alteração, faça um backup do seu arquivo de configuração do Netplan existente! Isso tornará mais fácil reverter se algo der errado: `sudo cp /etc/netplan/SEU_ARQUIVO_NETPLAN.yaml /etc/netplan/SEU_ARQUIVO_NETPLAN.yaml.bak`
*   **`netplan try`:** Use `sudo netplan try` *antes* de `sudo netplan apply`. `netplan try` aplicará a configuração temporariamente e reverterá automaticamente se detectar um problema. Isso pode evitar que você fique trancado para fora do seu sistema.

Esta abordagem Netplan é a maneira correta de configurar a rede no Ubuntu 24.04. Me avise se você tiver alguma dúvida!

## Criando VMs

Criar uma VM usando a linha de comando no KVM (usando `virt-install`) é uma ótima maneira de automatizar o processo ou criar VMs com configurações específicas. Aqui está um guia passo a passo:

**1. Pré-requisitos:**

*   **KVM e QEMU Instalados:** Certifique-se de que KVM e QEMU estejam instalados e funcionando corretamente. Você provavelmente já tem isso configurado se chegou até aqui.
*   **Imagem ISO:** Tenha a imagem ISO do sistema operacional que você deseja instalar na VM. Você precisará do caminho completo para este arquivo.
*   **Espaço em Disco:** Certifique-se de ter espaço em disco suficiente para a imagem da VM.

**2. O Comando `virt-install`:**

O comando básico `virt-install` tem a seguinte estrutura:

```bash
virt-install --name=NOME_DA_VM \
             --memory=TAMANHO_DA_MEMORIA \
             --vcpus=NUMERO_DE_CPUS \
             --os-variant=VARIANTE_DO_SO \
             --disk path=CAMINHO_PARA_IMAGEM_DE_DISCO,size=TAMANHO_DO_DISCO \
             --cdrom=CAMINHO_PARA_ISO \
             --network bridge=br0 \
             --graphics vnc,listen=0.0.0.0 \
             --console pty,target_type=serial \
             --noautoconsole
```

Vamos analisar cada opção:

*   `--name`: O nome da sua máquina virtual.
*   `--memory`: A quantidade de memória RAM para a VM (em MB). Exemplo: `2048` para 2GB.
*   `--vcpus`: O número de CPUs virtuais para a VM.
*   `--os-variant`: A "variante" do sistema operacional que você está instalando. Isso ajuda o KVM a otimizar a configuração. Você pode encontrar uma lista de variantes com o comando `osinfo-query os`. Exemplos: `ubuntu22.04`, `rhel8`, `debian11`. Se não encontrar uma variante exata, use uma variante mais genérica ou omita esta opção.
*   `--disk path`: O caminho para o arquivo de imagem de disco que será criado para a VM e o tamanho do disco em GB. Exemplo: `--disk path=/var/lib/libvirt/images/minha_vm.qcow2,size=20`.
*   `--cdrom`: O caminho para o arquivo ISO de instalação do sistema operacional.
*   `--network bridge`: Especifica a rede que a VM usará. Aqui, estamos usando a bridge `br0` que você configurou anteriormente.
*   `--graphics vnc,listen=0.0.0.0`: Configura o acesso gráfico à VM usando VNC. `listen=0.0.0.0` permite que você se conecte ao VNC de outras máquinas na rede.
*   `--console pty,target_type=serial`: Configura um console serial para acesso via linha de comando.
*   `--noautoconsole`: Impede que o `virt-install` tente se conectar automaticamente ao console da VM. Isso é útil quando você está usando VNC ou SSH.

**3. Exemplo Prático:**

Suponha que você queira criar uma VM com as seguintes características:

*   Nome: `ubuntu-server`
*   Memória: 2GB
*   CPUs: 2
*   Sistema Operacional: Ubuntu 22.04 Server
*   Disco: 20GB, localizado em `/var/lib/libvirt/images/ubuntu-server.qcow2`
*   ISO: `/home/seu_usuario/Downloads/ubuntu-22.04.3-live-server-amd64.iso`
*   Rede: `br0`

O comando `virt-install` seria:

```bash
virt-install --name=ubuntu-server \
             --memory=2048 \
             --vcpus=2 \
             --os-variant=ubuntu22.04 \
             --disk path=/var/lib/libvirt/images/ubuntu-server.qcow2,size=20 \
             --cdrom=/home/seu_usuario/Downloads/ubuntu-22.04.3-live-server-amd64.iso \
             --network bridge=br0 \
             --graphics vnc,listen=0.0.0.0 \
             --console pty,target_type=serial \
             --noautoconsole
```

**Substitua os valores no exemplo acima pelos seus valores reais!**

**4. Executar o Comando:**

Execute o comando no seu terminal. O `virt-install` irá criar a imagem de disco, configurar a VM e iniciar o processo de instalação.

**5. Conectar-se à VM:**

Como você está usando VNC, você precisará de um cliente VNC para se conectar à VM e completar a instalação do sistema operacional.

*   **Descobrir a Porta VNC:** Use o comando `virsh vncdisplay NOME_DA_VM` (substitua `NOME_DA_VM` pelo nome da sua VM) para descobrir a porta VNC atribuída à VM.  Por exemplo:

    ```bash
    virsh vncdisplay ubuntu-server
    ```

    A saída pode ser algo como `:1`. Isso significa que a porta VNC é 5901 (5900 + 1).

*   **Conectar com um Cliente VNC:** Use um cliente VNC (como Remmina, Vinagre, TigerVNC, etc.) para se conectar ao endereço IP do seu servidor KVM na porta VNC descoberta. Por exemplo, se o seu servidor KVM tem o IP `192.168.1.100` e a porta VNC é `5901`, você se conectaria a `192.168.1.100:5901`.

**6. Concluir a Instalação:**

Siga as instruções na tela dentro do cliente VNC para completar a instalação do sistema operacional na VM.

**Dicas e Solução de Problemas:**

*   **`osinfo-query os`:** Use este comando para obter uma lista de variantes de SO suportadas. Isso ajuda o KVM a otimizar a configuração da VM.
*   **Permissões:** Se você tiver problemas de permissão ao criar a imagem de disco, certifique-se de que o usuário `libvirt-qemu` tenha permissão para escrever no diretório onde você está criando a imagem.
*   **Rede:** Se a VM não conseguir se conectar à rede, verifique se a bridge `br0` está configurada corretamente e se o firewall não está bloqueando o tráfego.
*   **VNC:** Se você não conseguir se conectar ao VNC, verifique se o serviço VNC está em execução e se o firewall não está bloqueando a porta VNC.
*   **Automatização:** Para automatizar ainda mais o processo, você pode usar arquivos de resposta (kickstart para Red Hat/CentOS, preseed para Debian/Ubuntu) para automatizar a instalação do sistema operacional.

**O que é o `virt-viewer`?**

`virt-viewer` é uma ferramenta de console gráfico leve para máquinas virtuais (VMs). Ele se conecta ao console gráfico de uma VM que está sendo executada no KVM, QEMU, Xen ou outras plataformas de virtualização. Pense nele como um cliente VNC ou RDP específico para VMs, otimizado para funcionar bem com a infraestrutura de virtualização.

**Por que usar o `virt-viewer`?**

*   **Simplicidade:** É mais fácil de usar do que configurar manualmente um cliente VNC ou RDP. Ele descobre automaticamente a configuração da VM.
*   **Segurança:** Pode usar protocolos seguros como SPICE ou VNC sobre TLS.
*   **Desempenho:** Otimizado para VMs, oferecendo bom desempenho e recursos como redirecionamento de USB.
*   **Integração:** Integra-se bem com o `libvirt`, que é a camada de gerenciamento de virtualização comum no Linux.

**Como instalar o `virt-viewer`:**

A instalação é muito simples usando o gerenciador de pacotes da sua distribuição Linux.

*   **Ubuntu/Debian:**

    ```bash
    sudo apt update
    sudo apt install virt-viewer
    ```

*   **CentOS/RHEL/Fedora:**

    ```bash
    sudo dnf install virt-viewer
    ```

    (Ou `yum install virt-viewer` em versões mais antigas do CentOS/RHEL).

**Como usar o `virt-viewer`:**

1.  **Listar VMs:** Use o comando `virsh list` para ver uma lista das VMs em execução no seu sistema KVM. Anote o nome da VM que você deseja conectar.

    ```bash
    virsh list
    ```

2.  **Conectar à VM:** Execute o `virt-viewer` seguido pelo nome da VM:

    ```bash
    virt-viewer NOME_DA_VM
    ```

    Substitua `NOME_DA_VM` pelo nome real da sua VM.

    O `virt-viewer` se conectará automaticamente ao console gráfico da VM.

**Criando VMs no KVM (Revisão e Opções):**

Você tem várias maneiras de criar VMs no KVM:

1.  **`virt-install` (Linha de Comando):** Já cobrimos isso detalhadamente. É ótimo para automação e configurações precisas. Relembrando um exemplo:

    ```bash
    virt-install --name=minha_vm \
                 --memory=2048 \
                 --vcpus=2 \
                 --os-variant=ubuntu22.04 \
                 --disk path=/var/lib/libvirt/images/minha_vm.qcow2,size=20 \
                 --cdrom=/caminho/para/ubuntu.iso \
                 --network bridge=br0 \
                 --graphics vnc,listen=0.0.0.0 \
                 --noautoconsole
    ```

2.  **`virt-manager` (Interface Gráfica):** Uma interface gráfica mais amigável.

    *   **Instalação:**

        *   Ubuntu/Debian: `sudo apt install virt-manager`
        *   CentOS/RHEL/Fedora: `sudo dnf install virt-manager`

    *   **Uso:**
        1.  Abra o `virt-manager`.
        2.  Clique no botão "Create a new virtual machine".
        3.  Siga o assistente para configurar a VM (nome, memória, CPU, imagem de disco, ISO, rede, etc.).
        4.  O `virt-manager` criará a VM e iniciará o processo de instalação.

3.  **Definindo VMs a partir de XML (Avançado):**

    *   Você pode criar um arquivo XML que define todos os aspectos da sua VM (hardware, rede, armazenamento, etc.).
    *   Use o comando `virsh define arquivo.xml` para criar a VM a partir do arquivo XML.
    *   Isso oferece o máximo de flexibilidade, mas requer um conhecimento mais profundo do KVM e do formato XML.

## Comandos e conexão

O `virsh` é a ferramenta de linha de comando para gerenciar VMs no KVM usando o `libvirt`. Aqui estão alguns dos comandos mais úteis:

*   **`virsh list [--all]`:** Lista as VMs. Sem a opção `--all`, mostra apenas as VMs em execução. Com `--all`, mostra todas as VMs (em execução, desligadas, pausadas, etc.).

    ```bash
    virsh list
    virsh list --all
    ```

*   **`virsh start NOME_DA_VM`:** Inicia uma VM que está desligada.

    ```bash
    virsh start minha_vm
    ```

*   **`virsh shutdown NOME_DA_VM`:** Desliga uma VM de forma limpa, enviando um sinal de desligamento para o sistema operacional guest.

    ```bash
    virsh shutdown minha_vm
    ```

*   **`virsh destroy NOME_DA_VM`:** Desliga a VM imediatamente, sem enviar um sinal de desligamento (equivalente a desligar a energia). Use com cautela!

    ```bash
    virsh destroy minha_vm
    ```

*   **`virsh suspend NOME_DA_VM`:** Suspende a VM, salvando o estado atual na memória para o disco.

    ```bash
    virsh suspend minha_vm
    ```

*   **`virsh resume NOME_DA_VM`:** Retoma uma VM suspensa.

    ```bash
    virsh resume minha_vm
    ```

*   **`virsh console NOME_DA_VM`:** Conecta ao console da VM (se configurado).

    ```bash
    virsh console minha_vm
    ```

*   **`virsh dominfo NOME_DA_VM`:** Exibe informações detalhadas sobre a VM (estado, CPU, memória, disco, rede, etc.).

    ```bash
    virsh dominfo minha_vm
    ```

*   **`virsh edit NOME_DA_VM`:** Abre o arquivo XML de configuração da VM para edição. Use com extrema cautela, pois erros na configuração XML podem impedir que a VM inicie.

    ```bash
    virsh edit minha_vm
    ```

*   **`virsh autostart NOME_DA_VM`:** Define se a VM deve iniciar automaticamente na inicialização do sistema host.

    ```bash
    virsh autostart minha_vm  # Habilita o autostart
    virsh autostart --disable minha_vm  # Desabilita o autostart
    ```

*   **`virsh undefine NOME_DA_VM`:** Remove a definição da VM (não exclui os arquivos de disco).

    ```bash
    virsh undefine minha_vm
    ```

**Conexões SSH:**

SSH (Secure Shell) é um protocolo para acessar remotamente um sistema. Para se conectar a uma VM via SSH, você precisa:

1.  **Ter um servidor SSH instalado e em execução na VM.** A maioria das distribuições Linux já vem com um servidor SSH instalado, mas você pode precisar habilitá-lo. Por exemplo, no Ubuntu: `sudo systemctl enable ssh && sudo systemctl start ssh`
2.  **Conhecer o endereço IP da VM.** Você pode descobrir o IP da VM usando o comando `ip addr` dentro da VM, ou consultando o servidor DHCP do seu roteador.
3.  **Usar um cliente SSH para se conectar.** No Linux ou macOS, você pode usar o comando `ssh`:

    ```bash
    ssh usuario@endereco_ip_da_vm
    ```

    Substitua `usuario` pelo nome de usuário na VM e `endereco_ip_da_vm` pelo endereço IP da VM.
    Você será solicitado a inserir a senha do usuário.

    *   **Exemplo:**

        ```bash
        ssh ubuntu@192.168.1.101
        ```

    *   **Chaves SSH (Recomendado para Segurança):** Em vez de usar senhas, é altamente recomendável configurar chaves SSH para autenticação. Isso é mais seguro e conveniente.

**Comandos de Edição de Texto:**

*   **`nano`:** Um editor de texto simples e fácil de usar na linha de comando. Já o usamos bastante!

    ```bash
    nano arquivo.txt
    ```

*   **`vim` (ou `vi`):** Um editor de texto poderoso e flexível, mas com uma curva de aprendizado maior.

    ```bash
    vim arquivo.txt
    ```

*   **`gedit`:** Um editor de texto gráfico simples e amigável. Ele precisa de um ambiente gráfico para ser executado (então, você precisaria estar conectado ao servidor via SSH com X forwarding ou estar trabalhando diretamente no console do servidor).

    ```bash
    gedit arquivo.txt
    ```

**`virsh vcpupin`:**

Este comando permite "fixar" CPUs virtuais de uma VM a CPUs físicas específicas no host. Isso pode melhorar o desempenho em algumas situações, especialmente em cargas de trabalho sensíveis à latência.

*   **Sintaxe:**

    ```bash
    virsh vcpupin NOME_DA_VM VCPU CPU
    ```

    *   `NOME_DA_VM`: O nome da VM.
    *   `VCPU`: O número da CPU virtual (começa em 0).
    *   `CPU`: O número da CPU física no host.

*   **Descobrir CPUs Físicas:** Use o comando `lscpu` para obter informações sobre as CPUs físicas no seu sistema host.

*   **Exemplo:**

    Para fixar a VCPU 0 da VM "minha_vm" à CPU física 2:

    ```bash
    virsh vcpupin minha_vm 0 2
    ```

## Migração

Para migrar máquinas virtuais (VMs) entre diferentes hosts KVM ou até mesmo entre diferentes hipervisores, você pode utilizar métodos que envolvem exportação/importação, cópia de discos e configuração de rede. Aqui estão as principais abordagens:

---

### 1. **Migração ao vivo (Live Migration)**  
- **Recomendado se os hosts estiverem na mesma rede e com armazenamento compartilhado ou recursos de armazenamento comuns.**  
- Requer configuração de armazenamento compartilhado (como NFS, iSCSI, Ceph) e rede apropriada entre os hosts.  
- Utiliza comandos como:
  ```bash
  virsh migrate --live VMnome qemu+ssh://host2/system
  ```
- Precisa de configuração prévia de libvirt e autenticação SSH ou TLS.

---

### 2. **Migração offline (desligada)**  
- **Mais simples e compatível com diferentes ambientes ou quando armazenamento não é compartilhado.**  
- Basiamente, você faz o seguinte:
  - Exporta os discos da VM.
  - Transfere os discos para o novo host.
  - Cria uma nova VM no novo host apontando para os discos transferidos.

---

### 3. **Método passo a passo para migração offline (sem armazenamento compartilhado):**

#### no host de origem:
1. **Pare a VM (se estiver ligada):**
   ```bash
   virsh shutdown VMnome
   ```
2. **Exporte os discos virtuais (ex. qcow2):**
   ```bash
   scp /caminho/para/discos/VMnome.qcow2 usuario@host2:/destino/
   ```

#### no host destino:
3. **Crie uma nova VM (usando o XML da VM original ou via virt-manager):**
   - Você pode exportar o XML da VM original:
     ```bash
     virsh dumpxml VMnome > VMnome.xml
     ```
   - Edite o arquivo XML, se necessário, para ajustar paths de disco.
   
4. **Importe a VM na nova máquina:**
   ```bash
   virsh define VMnome.xml
   ```

---

### 4. **Migração entre diferentes hipervisores (como KVM para Xen, VMware, etc.)**
- O método mais comum é:
  - Converter discos para um formato universal (como QCOW2, VMDK, etc.).
  - Criar uma nova VM compatível com o hipervisor de destino.
  - Inserir os discos transferidos.
- Ferramentas como `qemu-img` ajudam na conversão:
  ```bash
  qemu-img convert -f qcow2 -O vmdk disco.qcow2 disco.vmdk
  ```
- Você precisará também montar ou editar a configuração da VM para que seja compatível com o novo ambiente.

---

### Resumindo:
- **Para migração ao vivo**: configure migração ao vivo no libvirt.
- **Para migração offline**: copie discos e crie uma nova VM no destino.
- **Para diferentes hipervisores**: converta discos e recrie a VM compatível.


## Endereçamento IP

Trabalhar com os IPs e redes das VMs, bem como conectar as VMs entre si, envolve configurar corretamente os tipos de rede e atribuir IPs de forma adequada. Aqui estão as principais opções e dicas:

---

### Tipos de redes para VMs no KVM:

1. **NAT (Network Address Translation)**  
   - As VMs obtêm IPs privados por DHCP, usando uma rede NAT gerenciada pelo libvirt (`virbr0`).  
   - **Boa para isolamento, acesso à internet, mas as VMs não se comunicam facilmente entre si ou com o host, a menos que configurado.**

2. **Rede ponte (bridge)**  
   - As VMs conectam-se à mesma ponte que o host, geralmente usando a ponte (`br0`).  
   - As VMs terão IPs na mesma rede do seu roteador/infraestrutura, podendo se comunicar entre si e com o host facilmente.  
   - Ideal para ambientes de servidores ou lugares onde as VMs precisam de IPs acessíveis na rede local.

3. **Rede interna (isolada)**  
   - Redes internas criadas pelo libvirt que só funcionam entre as VMs, sem acesso externo.  
   - Usada para comunicação exclusiva entre VMs.

---

### Como configurar redes IP e entre as VMs:

#### 1. **Definir uma rede em modo ponte ( preferência para comunicação entre VMs e acesso à rede externa):**

- Crie ou edite uma bridge (`br0`) como mostrei anteriormente.
- Configure suas VMs para usar essa ponte, via XML ou `virt-manager`.

#### 2. **Atribuição de IPs nas VMs:**

- **Via DHCP:**  
  - Use um servidor DHCP na sua rede (roteador, ou configure um DHCP a parte) para atribuir IPs automaticamente às VMs.
  
- **Manual (IP fixo):**  
  - Configure dentro da VM (Linux ou Windows) o IP fixo, máscara, gateway, DNS.

#### 3. **Para comunicação direta entre as VMs:**

- Certifique-se de que elas estejam na mesma rede (mesmo endereço de rede IP, máscara, etc.).
- Teste com `ping` entre as VMs.
- Caso esteja usando NAT ou redes internas, é só configurar IPs na mesma faixa.

---

### Resumo passo a passo:

1. Escolha o método de rede para as suas VMs (ponte é o mais comum para acessibilidade geral e comunicação entre elas).
2. Configure a rede (que pode envolver criar uma ponte no host).
3. Defina a rede no XML ou na interface do `virt-manager`.
4. Configure IPs nas VMs (DHCP ou IP fixo).
5. Teste a comunicação com `ping` ou outros meios.

---

## Backups

### 1. **Backup de uma VM**

**Opção simples:** Copiar os discos da VM e o XML de configuração.

**Passo a passo:**

- **Exportar a configuração XML:**
  ```bash
  virsh dumpxml nome_da_vm > nome_da_vm.xml
  ```
- **Copiar os discos da VM:**  
  Localize os discos (normalmente `.qcow2`, `.img`, `.raw`) usados pela VM e copie-os. Exemplo:
  ```bash
  cp /caminho/para/disco.qcow2 /backup/disco.qcow2
  ```
  Ou use `rsync` para transferir para um backup externo:
  ```bash
  rsync -avz /caminho/para/disco.qcow2 user@backup-server:/destino/
  ```

### 2. **Restore de uma VM**

**Para recuperar a VM a partir do backup:**

- Coloque os discos copiados na localização correta.
- Reimporte a VM com o XML:
  ```bash
  virsh define nome_da_vm.xml
  ```
- Se necessário, ajuste o arquivo XML para apontar para os discos corretos (editable com um editor de texto).


## Comando `watch`

### O que faz?

O comando `watch` executa periodicamente um comando e exibe sua saída, permitindo monitorar mudanças em tempo real.

### Sintaxe básica:

```bash
watch [opções] comando
```

### Exemplos comuns:

- **Atualizar a lista de processos:**

```bash
watch ps aux
```

- **Monitorar uso de CPU e memória do seu sistema:**

```bash
watch free -h
```

- **Verificar o status de uma VM a cada 2 segundos:**

```bash
watch virsh list --all
```

- **Atualizar a saída de um comando a cada 5 segundos:**

```bash
watch -n 5 comando
```

### Dica:
- Para parar o `watch`, basta usar `Ctrl + C`.

---

Se desejar, posso montar exemplos específicos de backups ou explicar comandos `watch` mais avançados!

## Referencias