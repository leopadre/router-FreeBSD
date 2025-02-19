#  Configuração de Rede no FreeBSD 14.2

O trabalho envolve a configuração de uma rede utilizando NAT, DHCP e DNAT em um ambiente FreeBSD 14.2. O objetivo é criar uma subrede privada (10.0.0.0/16) a partir de uma rede de acesso principal (192.168.133.0/24), permitindo que dispositivos da rede interna tenham acesso à internet por meio de um roteador configurado com NAT.

Para isso, é necessário configurar o serviço DHCP para fornecer endereços IP dinamicamente aos dispositivos da LAN, garantindo conectividade automática. Além disso, é preciso definir regras no firewall PF para permitir a comunicação entre as redes, incluindo a configuração de DNAT para redirecionar o tráfego da porta 80 do roteador para a porta 8080 de uma máquina específica dentro da LAN.


# Arquivos

### `/usr/local/etc/dhcpd.conf`

Arquivo de configuração do servidor DHCP, onde foram definidas as regras de alocação de endereços IP dinâmicos na subrede 10.0.0.0/16 e a configuração de um gateway padrão.

```
# Tempo padrão de concessão de um IP (em segundos)
default-lease-time 600;  # Um IP alugado será válido por 600 segundos (10 minutos) por padrão.

# Tempo máximo de concessão de um IP (em segundos)
max-lease-time 7200;  # Um IP alugado pode ser válido por até 7200 segundos (2 horas) no máximo.

# STATIC IP FOR VM1
# Configuração de um endereço IP estático para a máquina virtual chamada "VM1"
host VM1 {
  hardware ethernet 08:00:27:88:e0:0f;  # Define o endereço MAC da VM1 (identificador único da placa de rede).
  fixed-address 10.0.0.150;             # Atribui o IP fixo 10.0.0.150 para a VM1.
}

# Configuração da sub-rede
subnet 10.0.0.0 netmask 255.255.0.0 {  # Define a sub-rede 10.0.0.0 com máscara 255.255.0.0 (16 bits).
  range 10.0.0.100 10.0.0.200;         # Intervalo de IPs que o servidor DHCP pode atribuir dinamicamente.
  option routers 10.0.0.16;            # Define o gateway padrão (roteador) para os clientes DHCP.
  option domain-name-servers 8.8.8.8;  # Define o servidor DNS que os clientes DHCP devem usar (neste caso, o DNS público do Google).
  option broadcast-address 10.0.255.255;  # Define o endereço de broadcast da sub-rede.
}
```

### `/etc/pf.conf`

Arquivo de configuração do firewall PF, onde foram definidas regras de NAT para roteamento da rede interna, DNAT para redirecionamento de porta 80 para 8080, e regras de firewall para controle do tráfego de entrada e saída.

```
# DEFINICAO DAS INTERFACES
ext_if="em0"  # Interface externa (conectada à internet)
int_if="ue0"  # Interface interna (conectada à rede local)

# DEFINICAO DE VAR
localnet="10.0.0.0/16"  # Define a rede local (10.0.0.0 a 10.0.255.255)
ip2="10.0.0.150"        # IP de uma máquina específica na rede local
iphost="192.168.133.14" # IP de um host específico com regras especiais
porta="8080"            # Porta usada para redirecionamento de tráfego

# NAT (Network Address Translation)
nat on $ext_if from $localnet to any -> ($ext_if:0)
# Traduz os IPs da rede local para o IP da interface externa ao sair para a internet.

# DNAT (Destination NAT)
rdr on $ext_if proto tcp from any to ($ext_if) port 80 -> $ip2 port $porta
# Redireciona tráfego TCP da porta 80 (interface externa) para a porta 8080 da máquina com IP $ip2.

# FIREWALL
# INTERNAL NETWORK -> ANY
pass in on $int_if from $localnet to any keep state
# Permite tráfego de entrada na interface interna, da rede local para qualquer destino, mantendo o estado da conexão.

pass out on $ext_if keep state
# Permite tráfego de saída na interface externa, mantendo o estado da conexão.

# HOST TO ANY IP ON ROUTER
pass in on $ext_if from $iphost to any keep state
# Permite tráfego de entrada na interface externa, do host específico ($iphost) para qualquer destino, mantendo o estado.

pass out on $ext_if from any to $iphost keep state
# Permite tráfego de saída na interface externa, de qualquer origem para o host específico ($iphost), mantendo o estado.

# DNAT to maquina conectada
pass in on $ext_if proto tcp from any to ($ext_if) port $porta keep state
# Permite tráfego de entrada na interface externa, TCP destinado à porta $porta (8080), mantendo o estado.

pass out on $int_if proto tcp from ($ext_if) to $ip2 port $porta keep state
# Permite tráfego de saída na interface interna, TCP destinado ao IP $ip2 na porta $porta (8080), mantendo o estado.
```

### `/etc/rc.conf`

Arquivo de configuração do sistema, onde foram ativados o firewall PF e o servidor DHCP, e configuradas as interfaces de rede.

```
hostname="host"
keymap="br.kbd"
ifconfig_em0="DHCP"
ifconfig_em0_ipv6="inet6 accept_rtadv"
sshd_enable="YES"
moused_nondefault_enable="NO"
# Set dumpdev to "AUTO" to enable crash dumps, "NO" to disable
dumpdev="AUTO"

ifconfig_ue0="inet 10.0.0.16 netmask 255.255.0.0" # Configura a interface de rede
gateway_enable="YES"

# PF CONFIG
pf_enable="YES"
pf_rules="/etc/pf.conf"
pf_flags=""
pflog_enable="YES"
pflog_logfile="/var/log/pflog"
pflog_flags=""
pflogd_enable="YES"
pfsync_enable="YES"

# DHCP
dhcpd_enable="YES"
dhcpd_ifaces="ue0"
```

## Configurando o Servidor

### Instalação do Servidor DHCP

```
pkg install isc-dhcp44-server
```

Instala o servidor DHCP no FreeBSD.

### Ativando o Encaminhamento de Pacotes

```
sysctl net.inet.ip.forwarding=1
```

Habilita o roteamento de pacotes entre interfaces de rede.

### Ativa o Arquivo PF(Packet Filter)

```
service pf start
```

### Aplicando as Regras do Firewall

```
pfctl -f /etc/pf.conf
```

Recarrega as regras definidas no arquivo `/etc/pf.conf`.

```
pfctl -e
```

### Iniciando o Servidor DHCP

```
service isc-dhcpd start
```

Inicia o serviço DHCP para fornecer endereços IP dinâmicos aos clientes da rede.

### Reinicializando Interfaces de Rede

```
service netif restart
```

Reinicia as interfaces de rede para aplicar as novas configurações de IP.

### Adendo

Caso as configurações não funcionem, reinicie o sistema. Se o arquivo `/etc/rc.conf` estiver devidamente configurado, as ativações necessárias serão feitas automáticamente.

## Testando a Configuração

Para garantir que a configuração foi aplicada corretamente, siga os passos abaixo:

### Teste de Atribuição de IP Dinâmico

1.  Em um dispositivo conectado à LAN, verifique se ele recebeu um IP do servidor DHCP:
    
    ```
    ifconfig
    ```
    
    O dispositivo deve receber um IP dentro do intervalo definido no arquivo `dhcpd.conf` para o caso da atribuição dinâmica, ou o IP estático definido.

### Teste de Redirecionamento de Porta (DNAT)

1.  No cliente conectado ao servidor, inicie um listener na porta 8080:
    
    ```
    nc -lvnp 8080
    ```
    
2.  Em outra maquina da rede WAN, tente conectar-se ao roteador na porta 80:
    
    ```
    nc -vn 192.168.133.0 80
    ```
    
    Se a configuração estiver correta, a conexão será estabelecida e será possível trocar mensagens entre as máquinas.
    

### Teste de Conectividade

1.  Dê um ping no servidor DNS público do Google:
    
    ```
    ping 8.8.8.8
    ```
    
    ou
    
    ```
    ping google.com
    ```
    
    Se a resposta for recebida, significa que o servidor está roteando corretamente os pacotes para a LAN. A conectividade também pode ser testada simplesmente acessando alguma página da internet.


### Restrições

- A segurança da configuração de rede é baixa considerando que todas as portas estão abertas e responsivas, uma configuração que usamos apenas para depuração e não deve ser uma solução de produção
- Gerenciar hosts que recebem IPs estáticos é muito manual com a necessidade de abrir o arquivo de configuração e reinicializar o sistema toda vez
- Nenhum gerenciamento sem fio é garantido, pois nenhum ponto de acesso foi disponibilizado para teste
