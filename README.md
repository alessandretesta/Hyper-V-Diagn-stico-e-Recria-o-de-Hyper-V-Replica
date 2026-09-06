# Hyper-V-Diagn-stico-e-Recria-o-de-Hyper-V-Replica
Estudar o diagnóstico de falhas de replicação e o processo de recriação de uma réplica de máquina virtual no Hyper-V.
📘 Caderno de Estudos: Engenharia, Arquitetura e Administração de Hyper-V
Este repositório serve como um guia de referência completo e prático para o estudo e a administração de ambientes virtualizados utilizando o Microsoft Hyper-V [139, 479]. Ele reúne conceitos estruturais detalhados de arquitetura, guias operacionais passo a passo e scripts prontos para automação em PowerShell [33, 140, 581, 618].
---
📌 1. Resumo Estruturado do Hyper-V
O Hyper-V é um hipervisor Tipo 1 (bare-metal) de classe empresarial desenvolvido pela Microsoft, integrado nativamente às versões Pro, Enterprise e Education do Windows 10/11 e ao Windows Server [18, 139, 248, 479]. Ele roda diretamente no hardware físico, oferecendo excelente isolamento de processos, alta performance e resiliência contínua para cargas de trabalho corporativas [139, 479].
Arquitetura de Microkernel e Isolamento
A arquitetura do Hyper-V baseia-se em um microkernel enxuto que gerencia apenas recursos básicos de processamento e memória [140, 403]. Ele não possui drivers de terceiros integrados ao seu núcleo [140]. Em vez disso, o controle operacional é executado através de partições lógicas [140, 514]:
Partição Pai (Root Partition): A primeira partição a ser inicializada, executando uma versão de 64 bits do Windows [141, 514]. Ela detém acesso direto aos recursos físicos de E/S e gerencia as partições filhas por meio da pilha de gerenciamento de virtualização e das APIs de hypercalls [141, 514].
Partições Filhas (Child Partitions): Onde são hospedadas as máquinas virtuais convidadas (Guests) [141, 514]. Elas não acessam o hardware físico diretamente; interagem com dispositivos virtuais sintetizados e emulados fornecidos pela partição pai [141, 515].
```
┌────────────────────────────────────────────────────────┐
│              Partição Pai (Root/Parent)                │
│  ┌──────────────────────┐   ┌───────────────────────┐  │
│  │ Virtualization Stack │   │  Drivers de Hardware  │  │
│  └──────────────────────┘   └───────────────────────┘  │
└───────────────────────────┬────────────────────────────┘
                            │ VMBus (Inter-partition) [142, 516]
┌───────────────────────────▼────────────────────────────┐
│              Partições Filhas (Child/VMs)              │
│  ┌──────────────────────┐   ┌───────────────────────┐  │
│  │   OS Convidado       │   │    Drivers Sintéticos │  │
│  │   (Windows / Linux)  │   │    (VSCs / NetVSC)    │  │
│  └──────────────────────┘   └───────────────────────┘  │
└────────────────────────────────────────────────────────┘
```
Tecnologias de Discos Virtuais
O planejamento do armazenamento no Hyper-V exige compreender as diferentes opções de arquivos de disco [145]:
VHD: Formato legado, limitado a 2 TB por arquivo e desprovido de salvaguardas robustas contra corrupção por perda súbita de energia [146, 757, 758].
VHDX: Padrão moderno que estende a capacidade para até 64 TB [25, 146, 435, 758]. Possui proteção nativa de metadados por journaling, alinhamento automático para setores físicos de 4 KB (mitigando a amplificação de escrita) e otimização para blocos grandes [14, 146, 147, 435, 436].
VHD Set (.VHDS): Modelo de disco compartilhado lançado no Windows Server 2016 para guest clustering [106, 146]. Permite checkpoints consistentes com aplicativos, replicação assíncrona e redimensionamento online de discos compartilhados [106, 146].
Alinhamento e Concorrência de E/S
O desempenho de armazenamento é altamente afetado pelo alinhamento do disco [147]. Discos desalinhados forçam a controladora física a ler e gravar múltiplos blocos físicos para cada bloco virtual lógico, gerando latência drástica [147]. Discos criados no Windows Server 2012 ou posterior possuem alinhamento automático de 4 KB (comprovado pela propriedade `Alignment = 1` no cmdlet `Get-VHD`) [147, 430, 432, 434].
Para alta concorrência de E/S em servidores com alta demanda (como bancos de dados e servidores de correio), recomenda-se o uso de múltiplas controladoras SCSI sintéticas [149, 158].
Gerenciamento de Memória Dinâmica vs. Virtual NUMA (vNUMA)
Virtual NUMA: Expõe a topologia física de nós NUMA do processador para dentro da máquina virtual, otimizando o acesso à memória RAM local e minimizando os custos de latência em barramentos remotos de CPU [153, 415, 416]. É indispensável para aplicações sensíveis, como o Microsoft SQL Server [154, 158, 417].
Memória Dinâmica: Ajusta automaticamente a RAM da VM sob demanda [25, 40, 249, 758]. No entanto, ela inviabiliza a topologia vNUMA, nivelando a máquina virtual sob um único nó virtual plano [417].
⚠️ Regra de Ouro: Desative a Memória Dinâmica para cargas de alta transação (como bancos de dados SQL Server, Microsoft Exchange Server e Controladores de Domínio Active Directory) para mitigar quedas de desempenho e exaustão de buffers [9, 13, 19, 20, 155, 156, 158].
Alta Disponibilidade e Resiliência
Failover Clustering: Tecnologia que une nós físicos em um cluster para garantir alta disponibilidade [170, 237, 244]. Utiliza batimentos cardíacos (heartbeats) contínuos e requer configurações estritas de Quorum (como Node Majority ou testemunhas de disco/compartilhamento) para impedir cenários de partição de rede (split-brain) [170, 238, 244].
Cluster Shared Volumes (CSV): Abstração que permite que múltiplos nós do cluster tenham acesso concorrente de leitura/gravação ao mesmo LUN formatado em NTFS ou ReFS, facilitando migrações online (Live Migration) ultrarrápidas sem interrupção de conexões [5, 151, 526].
Hyper-V Replica: Solução nativa de Disaster Recovery que realiza replicação assíncrona periódica de alterações acumuladas de arquivos de discos VHDX para hosts ou clusters secundários (réplicas) em intervalos configuráveis de 30 segundos, 5 minutos ou 15 minutos [84, 328, 330]. Dispensa armazenamento compartilhado [329].
---
📖 2. Índice e Glossário (Foco em Vídeos)
Este glossário detalha os principais termos e conceitos operacionais e de infraestrutura abordados nas fontes de vídeo do caderno:
Host (Hospedeiro): O computador ou servidor físico que executa o sistema de virtualização Hyper-V e fornece recursos de CPU, memória, rede e armazenamento para as instâncias virtuais [18, 141, 753].
Guest (Convidado): A máquina virtual (VM) hospedada no host que executa seu próprio sistema operacional convidado de forma isolada [18, 141, 753].
Partição Pai (Root Partition): A única partição lógica executada no hipervisor com privilégios de acesso direto aos drivers e ao hardware físico, hospedando a pilha de virtualização e os serviços de gerenciamento [141, 514, 518].
Partição Filha (Child Partition): Qualquer máquina virtual cliente criada e isolada pela partição pai, que recebe recursos de hardware em formato de representação virtualizada (VDevs) [141, 467, 514, 518].
VMBus (Virtual Machine Bus): Barramento lógico de comunicação interpartição de alta velocidade. Ele interliga os clientes sintéticos das VMs (VSCs) aos provedores de serviços da partição pai (VSPs) [142, 472, 516, 518].
VSP (Virtualization Service Provider): Serviço executado na partição pai que processa as requisições de E/S físicas provenientes das partições filhas pelo barramento VMBus [142, 471, 516, 518].
VSC (Virtualization Service Client): Driver sintético instalado na partição filha que consome os serviços disponibilizados pelos VSPs no host sem sofrer gargalos de emulação de hardware [142, 471, 516, 518].
VHDX: Formato moderno de disco virtual utilizado pelo Hyper-V [25, 146, 758]. Suporta discos de até 64 TB, é resiliente a corrupções por falta de energia devido ao journaling de metadados e oferece melhor desempenho em mídias físicas de setores grandes [13, 25, 146, 435, 758].
Expansão Dinâmica: Tipo de alocação de disco virtual onde o arquivo no host cresce gradualmente de tamanho conforme novos arquivos são gravados dentro da VM, reduzindo o desperdício de espaço no host [25, 37, 280, 759].
Tamanho Fixo: Tipo de alocação de disco virtual em que o tamanho total definido (ex: 100 GB) é totalmente reservado e alocado imediatamente no host física, oferecendo menor fragmentação e maior desempenho de gravação [25, 37, 280, 758].
Comutador Virtual Externo (External Switch): Rede virtual que se vincula diretamente a uma placa de rede física do host, permitindo que as VMs acessem a rede local física e a Internet, comportando-se como nós reais da rede física [24, 28, 506, 601].
Comutador Virtual Interno (Internal Switch): Rede que possibilita a comunicação interna apenas entre as VMs e entre as VMs e a própria máquina host [24, 28, 506, 601].
Comutador Virtual Particular (Private Switch): Rede totalmente isolada na qual as VMs conseguem se comunicar somente entre si, sem visibilidade da máquina host ou de qualquer rede física externa [24, 28, 506, 601].
Sessão Avançada (Enhanced Session Mode): Recurso que melhora a integração entre o host e a VM, habilitando redirecionamento de áudio, impressoras, dispositivos locais, compartilhamento de área de transferência (copiar e colar arquivos) e ajuste dinâmico de resolução de tela por meio da pilha do Remote Desktop (RDP) [51, 52, 73, 276, 303]. Exige sistemas de nível Professional ou superior [52, 73, 276, 306].
Pontos de Verificação (Checkpoints): Capturas de estado que "congelam" o momento atual da VM (criando discos diferenciais), úteis para isolar testes de software e permitir a reversão rápida do sistema operacional convidado para um estado anterior [45, 266, 280].
Atribuição de Dispositivo Discreto (DDA / Discrete Device Assignment): Recurso que permite que uma máquina virtual contorne o switch virtual e tome controle físico direto de periféricos PCIe do host, como placas de vídeo de alta performance (GPUs) ou controladoras NVMe [27, 188].
Redirecionamento de Dispositivos USB via RemoteFX: Diretiva avançada de grupo que possibilita redirecionar dispositivos USB complexos conectados fisicamente no host diretamente para serem lidos e configurados dentro de sessões RDP da VM [304, 306, 307].
---
💻 3. Automação e Administração via PowerShell
Os comandos abaixo consolidam os principais prompts e scripts em PowerShell necessários para realizar o ciclo de vida completo de uma máquina virtual, sua configuração de rede com acesso à internet e o estabelecimento de uma arquitetura de replicação.
> **💡 Requisito Prévio:** Todos os comandos devem ser executados em um terminal PowerShell como **Administrador** [33, 147, 521].
3.1. Ativação e Preparação do Ambiente
Para habilitar o recurso do Hyper-V no sistema e configurar o processador do host físico para suportar Virtualização Aninhada (Nested Virtualization) para uma VM específica [33, 136, 164, 521]:
```powershell
# 1. Habilitar o recurso Hyper-V e ferramentas de gerenciamento no Windows (Requer Reinicialização)
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All

# 2. Habilitar Virtualização Aninhada em uma VM específica (deve ser executado com a VM DESLIGADA)
Set-VMProcessor -VMName "SuaVMHost" -ExposeVirtualizationExtensions $true
```
3.2. Criação da Máquina Virtual do Zero (Passo a Passo)
Este bloco realiza a criação de um disco rígido virtual de expansão dinâmica (`.vhdx`) [25, 37, 758] e configura uma máquina virtual de Geração 2 de alta performance [55, 68]:
```powershell
# 1. Variáveis de Ambiente
$VMName = "Ubuntu_Prod_01"
$Path = "C:\VMs"
$VHDPath = "$Path\$VMName\Virtual Hard Disks\$VMName.vhdx"
$ISOPath = "C:\ISO\ubuntu-server-20.04.iso"

# 2. Criar a pasta base para a máquina virtual
New-Item -ItemType Directory -Path "$Path\$VMName"

# 3. Criar o disco virtual VHDX dinamicamente expansível (100 GB)
New-VHD -Path $VHDPath -SizeBytes 100GB -Dynamic

# 4. Criar a máquina virtual de Geração 2 (UEFI) com 4 GB de RAM estática
New-VM -Name $VMName -Generation 2 -MemoryStartupBytes 4096MB -Path $Path

# 5. Associar o disco rígido criado à controladora SCSI da VM
Add-VMHardDiskDrive -VMName $VMName -Path $VHDPath -ControllerNumber 0 -ControllerLocation 0

# 6. Adicionar unidade de DVD e anexar a imagem ISO de instalação
Add-VMDvdDrive -VMName $VMName -Path $ISOPath -ControllerNumber 0 -ControllerLocation 1

# 7. Configurar os núcleos do Processador virtual (ex: 4 núcleos)
Set-VMProcessor -VMName $VMName -Count 4
```
3.3. Configurando Acesso à Internet (Modo Bridge / Switch Externo)
Para criar um comutador de rede virtual externo ligado ao adaptador de rede físico real do seu host e conectar o adaptador da VM [24, 28, 506, 601]:
```powershell
# 1. Identificar o nome exato do adaptador de rede físico conectado do seu Host
Get-NetAdapter | Format-Table -Property Name, InterfaceDescription, Status

# 2. Criar o Switch Virtual Externo (Substitua "Ethernet" pelo nome do adaptador obtido no passo 1)
New-VMSwitch -Name "Switch_Externo_Prod" -NetAdapterName "Ethernet" -AllowManagementOS $true -MinimumBandwidthMode Weight

# 3. Adicionar o adaptador de rede virtual à VM e associá-lo ao switch externo
Add-VMNetworkAdapter -VMName "Ubuntu_Prod_01" -SwitchName "Switch_Externo_Prod" -Name "NIC01"

# 4. Configurar peso de qualidade de serviço (QoS) para o tráfego da VM
Set-VMNetworkAdapter -VMName "Ubuntu_Prod_01" -Name "NIC01" -MinimumBandwidthWeight 10
```
3.4. Implementação Completa do Hyper-V Replica
Para construir um fluxo de replicação assíncrona robusto entre dois servidores que fazem parte do mesmo Active Directory (Kerberos/HTTP na Porta 80) [88, 633]:
Passo 1: Configuração de Firewall e Serviço no Servidor de Réplica (Destino)
Este passo configura o firewall local no servidor que receberá as alterações e ativa a escuta de réplica [341, 342, 344]:
```powershell
# 1. Habilitar a regra de firewall para aceitar tráfego de entrada HTTP na porta 80
Enable-NetFirewallRule -DisplayName "Ouvinte de HTTP da Réplica do Hyper-V (TCP-In)"

# 2. Configurar o computador para agir como servidor de Réplica
# Permite tráfego HTTP básico por Kerberos e aceita conexões de qualquer host do domínio
Set-VMReplicationServer -ReplicationEnabled $true -AllowedReceiverType Kerberos -Port 80 -DefaultStorageType Local -DefaultStoragePath "C:\VMs\Replicas"
```
Passo 2: Configuração e Início de Replicação no Servidor Primário (Origem)
Habilita a replicação para a VM desejada e inicia o envio inicial de dados pela rede física [633, 634]:
```powershell
# 1. Habilitar a replicação assíncrona na VM primária direcionando para o host de réplica
# Frequência definida para 5 minutos (300 segundos) com compactação de tráfego ativa
Enable-VMReplication -VMName "Ubuntu_Prod_01" -ReplicaServerName "REPLICA_HOST_FQDN" -ReplicaServerPort 80 -AuthenticationType Kerberos -CompressionEnabled $true -FrequencySec 300

# 2. Inicializar o envio imediato da réplica inicial de blocos pela rede
Start-VMInitialReplication -VMName "Ubuntu_Prod_01"
```
Passo 3: Comandos de Monitoramento de Integridade e Sincronização
Para acompanhar o andamento da réplica e a taxa de transferência [649, 650]:
```powershell
# Verificar o estado atual e integridade da replicação
Get-VMReplication -VMName "Ubuntu_Prod_01" | Format-List VMName, State, Health, Mode, PrimaryServer, ReplicaServer

# Medir as estatísticas de latência e tamanho médio de envio das replicações
Measure-VMReplication -VMName "Ubuntu_Prod_01"
```
---
🛠️ 4. Guia de Resolução de Problemas (Troubleshooting)
Quando a replicação ou o estado das VMs apresentar falhas, utilize as seguintes diretrizes para rápida resolução [668]:
🛑 Replicação com Status "Critical" (Crítico)
Isso indica que as alterações pararam de ser enviadas ou acumulou-se um log de diferenças impossível de sincronizar [645].
Conectividade: Tente forçar a retomada manual [133]:
    ```powershell
    Resume-VMReplication -VMName "Ubuntu_Prod_01"
    ```
Resincronização: Se a VM ficou desligada ou inacessível por muito tempo, os dados divergiram e exigem uma resincronização [133]:
    ```powershell
    Start-VMFailover -VMName "Ubuntu_Prod_01" -Resynchronize
    ```
Remoção de Pontos de Verificação (Checkpoints): Se necessário em desastres, remova checkpoints antigos para liberar espaço em disco e permitir a consolidação [214]:
    ```powershell
    Complete-VMFailover -VMName "Ubuntu_Prod_01"
    ```
🛑 Limpeza Completa para Refazer uma Replicação do Zero
Caso a replicação quebre totalmente e você queira reconfigurar tudo limpo do início [689, 691]:
No Servidor Primário: Clique com o botão direito na VM, vá em Replicação e selecione Remover Replicação (Status deve ficar como "Não Aplicável") [689].
No Servidor de Réplica:
Delete a máquina virtual de teste/placeholder do gerenciador [689, 690].
Importante: Acesse fisicamente o repositório de dados e remova a pasta de discos virtuais antiga (`C:\VMs\Replicas\Ubuntu_Prod_01`) [690]. Se houver arquivos residuais, o Hyper-V bloqueará o novo início de replicação [690].
Refazer: Execute os comandos descritos no subitem 3.4 novamente.
---
Este documento serve como a base de operação e guia de governança do seu ambiente virtualizado Hyper-V. Utilize os exemplos fornecidos de forma modular no seu laboratório ou infraestrutura de rede corporativa!
https://notebook.google.com/notebook/e60026e0-06e1-41dd-ac3d-fe031e3e97d8
