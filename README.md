#  Configuração de Rede no FreeBSD

O trabalho envolve a configuração de uma rede utilizando NAT, DHCP e DNAT em um ambiente FreeBSD. O objetivo é criar uma subrede privada (10.0.0.0/16) a partir de uma rede de acesso principal (192.168.133.0/24), permitindo que dispositivos da rede interna tenham acesso à internet por meio de um roteador configurado com NAT.

Para isso, é necessário configurar o serviço DHCP para fornecer endereços IP dinamicamente aos dispositivos da LAN, garantindo conectividade automática. Além disso, é preciso definir regras no firewall PF para permitir a comunicação entre as redes, incluindo a configuração de DNAT para redirecionar o tráfego da porta 80 do roteador para a porta 8080 de uma máquina específica dentro da LAN.


# Arquivos

### `/usr/local/etc/dhcpd.conf`

Arquivo de configuração do servidor DHCP, onde foram definidas as regras de alocação de endereços IP dinâmicos na subrede 10.0.0.0/16 e a configuração de um gateway padrão.

### `/etc/pf.conf`

Arquivo de configuração do firewall PF, onde foram definidas regras de NAT para roteamento da rede interna, DNAT para redirecionamento de porta 80 para 8080, e regras de firewall para controle do tráfego de entrada e saída.

### `/etc/rc.conf`

Arquivo de configuração do sistema, onde foram ativados o firewall PF e o servidor DHCP, e configuradas as interfaces de rede.


## Comandos Utilizados

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

### Aplicando as Regras do Firewall

```
pfctl -f /etc/pf.conf
```

Recarrega as regras definidas no arquivo `/etc/pf.conf`.

```
pfctl -e
```

Ativa o firewall PF.

### Reinicializando Interfaces de Rede

```
service netif restart
```

Reinicia as interfaces de rede para aplicar as novas configurações de IP.

### Iniciando o Servidor DHCP

```
service isc-dhcpd start
```

Inicia o serviço DHCP para fornecer endereços IP dinâmicos aos clientes da rede.

## Testando a Configuração

Para garantir que a configuração foi aplicada corretamente, siga os passos abaixo:

### Teste de Atribuição de IP Dinâmico

1.  Em um dispositivo conectado à LAN, verifique se ele recebeu um IP do servidor DHCP:
    
    ```
    ifconfig ue0
    ```
    
    ou
    
    ```
    dhclient ue0
    ```
    
    O dispositivo deve receber um IP dentro do intervalo definido no arquivo `dhcpd.conf`.
2.  Verifique o IP da máquina conectada, utilize:
    
    ```
    ifconfig ue0
    ```
    
    O endereço IP atribuído à interface `ue0` será exibido. 

### Teste de Redirecionamento de Porta (DNAT)

1.  No servidor conectado ao roteador, inicie um listener na porta 8080:
    
    ```
    nc -lvnp 8080
    ```
    
2.  Em outro computador da rede WAN, tente conectar-se ao roteador na porta 80:
    
    ```
    nc -vn 192.168.133.0 80
    ```
    
    Se a configuração estiver correta, a conexão será estabelecida e será possível trocar mensagens entre as máquinas.
    

### Teste de Conectividade

1.  Dê um ping na máquina conectada ao roteador para verificar a conectividade:
    
    ```
    ping 10.0.0.0
    ```
    
    Se a resposta for recebida, significa que o roteador está roteando corretamente os pacotes para a LAN.
