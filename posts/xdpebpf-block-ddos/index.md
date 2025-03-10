# XDP &amp; EBPF Block DoS



Segue um teste para aplicar os conceitos de eBPF! É um esboço de uma solução que espera ser funcional e que combina um programa [XDP](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/introduction.html#what-is-xdp)/[eBPF](https://ebpf.io/) em C com um script em Bash(antes de usar Go) para monitorar o tráfego e atualizar dinamicamente um [map](https://docs.cilium.io/en/latest/reference-guides/bpf/architecture/#maps) [eBPF](https://ebpf.io/) com sources IPs que devem ser bloqueados quando idenficado um possível ataque [DoS](https://attack.mitre.org/techniques/T0814/). 

- Nesse cenário, o programa XDP/eBPF já está instalado na interface de rede e bloqueia pacotes provenientes de IPs listados no [map](https://docs.cilium.io/en/latest/reference-guides/bpf/architecture/#maps). 
- O script em Bash monitora  o arquivo de subsistema de rastreamento [conntrack](https://conntrack-tools.netfilter.org) do [Netfilter](https://www.netfilter.org/) [/proc/net/nf_conntrack](https://conntrack-tools.netfilter.org/manual.html#conntrack) para determinar se o número de conexões ativas ultrapassou o threshold configurado, e que ao ser atingido:
	- um log gerado por regras específicas [nftables](https://www.netfilter.org/projects/nftables/index.html) é analisado para identificar com base em uma &#34;janela&#34; de tempo, qual source IP ultrapassou o limite de conexões: indicando ser um possível ataque DoS:
		- que então é inserido no `map` eBPF, fazendo com que o programa XDP passe a descartar os pacotes desses endereços.



## 1. Pré-requisitos

- Kernel Linux com suporte a [eBPF e XDP](https://docs.cilium.io/en/latest/reference-guides/bpf/index.html#bpf-and-xdp-reference-guide)  
    Certifique-se de utilizar uma distribuição com kernel moderno (&gt;= 4.18, de preferência 5.x ou superior).
- clang
- llvm
- gcc
- libbpf
- libbpf-devel 
- libxdp 
- libxdp-devel 
- xdp-tools 
- bpftool kernel-headers
- perf
- nftables
- iproute2


## Considrações e Limitações
- Rastreamento de Conexões:
	
  - **Real time**

  O `nf_conntrack` é atualizado dinamicamente pelo kernel, oferecendo um “instantâneo” do estado atual das conexões, possibilitando a detecção rápida de mudanças no tráfego.  
  - **Stateful**

  O `nf_conntrack` dá uma visão de conexões `stateful` ativas e seus estados (NEW, ESTABLISHED, RELATED e FIN_WAIT…). Cada linha em   `/proc/net/nf_conntrack` representa uma entrada que pode conter informações detalhadas, como endereços IP de origem/destino, portas, protocolos e o estado da conexão. Comprometendo a detecção de protocolos stateless... 
   - **Tabela conntrack**

  Tem tamanho limitado e, em condições normais, é dimensionada para o tráfego esperado. Mas em ataques volumétricos, o número de conexões satura a tabela, impedindo que novas conexões sejam registradas. Isso não só prejudica a detecção como pode impactar a disponibilidade do serviço, mesmo para tráfego legítimo.
  - **Falsos Positivos**

  Em ambientes com tráfego elevado – por exemplo, servidores com muitos clientes reais ou aplicações que abrem múltiplas conexões simultâneas – um pico pode ocorrer de forma legítima. Sem uma correlação com outros indicadores (como análise de logs ou métricas de latência), o sistema pode interpretar um aumento repentino de conexões como um ataque. 
  - **Complementaridade**

  Para uma detecção mais robusta, é recomendável combinar a análise de diversas fontes de dados. 
  - **Latência na Atualização**

  O método de contagem, que usa wc -l para contar linhas, pode não refletir exatamente a taxa de chegada de novas conexões, especialmente em sistemas de alta performance, onde as entradas podem ser criadas e removidas em milissegundos.

  O uso de um script em Bash para monitorar o arquivo de conntrack e processar logs introduzir latência. Em um cenário de ataque DoS, a rapidez na detecção e na inserção do IP no map é crucial para minimizar o impacto do ataque.
  
  A comunicação entre a camada de monitoramento em user space e o programa eBPF no kernel tem atrasos, permitindo que pacotes passem antes da inserção no map.

  O arquivo /proc/net/nf_conntrack pode não refletir com precisão o estado em tempo real das conexões, especialmente em ambientes com alto volume de tráfego.

  Basear a detecção apenas em logs e conexões pode gerar falsos positivos ou, inversamente, deixar passar ataques se a janela de tempo não for bem ajustada.

  Usar **tracepoints e perf events**  definitivamente é melhor, principalmente por estar em kernel space. Mas fica para outro post. Por agora, vale chamar a atenção para os recursos usados...


## 2. Programa XDP/eBPF

O programa utiliza um [map](https://docs.cilium.io/en/latest/reference-guides/bpf/architecture/#maps) eBPF do tipo HASH para armazenar os IPs que devem ser bloqueados. E para cada pacote recebido, é verificado se source IP consta no mapa. Se sim, o pacote é descartado ([XDP_DROP](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/implementation/xdp_actions.html)); caso contrário, ele é encaminhado normalmente ([XDP_PASS](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/implementation/xdp_actions.html)).

&gt;[!INFO] **RTFM** e código atualizado: [repo Git](https://github.com/0xttfx/xdp-block-dos)

---

```c
#include &lt;linux/bpf.h&gt;
#include &lt;linux/if_ether.h&gt;
#include &lt;linux/ip.h&gt;
#include &lt;bpf/bpf_helpers.h&gt;
#include &lt;bpf/bpf_endian.h&gt;

/*
 * Mapa do tipo HASH para armazenar os IPs bloqueados.
 * - Chave: __u32 (IP de origem, em formato de rede, em big-endian)
 * - Valor: __u32 (flag, por exemplo, 1 para indicar bloqueio)
 * - max_entries: define quantos IPs podem ser bloqueados simultaneamente
 */
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, __u32);
    __type(value, __u32);
} blocked_ips SEC(&#34;.maps&#34;);

/*
 * Programa XDP para bloquear pacotes provenientes de IPs listados no mapa.
 *
 * Fluxo do programa:
 *   1. Obtém os ponteiros para o início e fim dos dados do pacote.
 *   2. Valida a presença completa do cabeçalho Ethernet.
 *   3. Processa somente pacotes IPv4.
 *   4. Valida a presença do cabeçalho IP completo e a integridade do campo IHL.
 *   5. Extrai o IP de origem (localizado nos primeiros 20 bytes do cabeçalho IP).
 *   6. Consulta o mapa &#39;blocked_ips&#39; e, se o IP estiver bloqueado, retorna XDP_DROP.
 *      Caso contrário, o pacote segue normalmente (XDP_PASS).
 *
 * Detalhes de performance:
 *   - A execução ocorre no caminho de dados (driver) com alta performance e baixa latência.
 *   - As verificações de limites são essenciais para satisfazer o verificador do eBPF.
 *   - A lógica foi mantida mínima, sem operações adicionais, para otimizar o tempo de execução.
 */
SEC(&#34;xdp&#34;)
int xdp_block_dos(struct xdp_md *ctx)
{
    // Obtém os ponteiros para os dados do pacote
    void *data = (void *)(long)ctx-&gt;data;
    void *data_end = (void *)(long)ctx-&gt;data_end;

    // Verifica se o cabeçalho Ethernet está completo
    struct ethhdr *eth = data;
    if ((void *)(eth &#43; 1) &gt; data_end)
        return XDP_PASS;

    // Processa somente pacotes IPv4
    if (bpf_ntohs(eth-&gt;h_proto) != ETH_P_IP)
        return XDP_PASS;

    // Calcula o início do cabeçalho IP (logo após o cabeçalho Ethernet)
    struct iphdr *ip = data &#43; sizeof(*eth);
    
    // Verifica se pelo menos os 20 bytes mínimos do cabeçalho IP estão presentes.
    if ((void *)ip &#43; sizeof(struct iphdr) &gt; data_end)
        return XDP_PASS;

    // Valida o campo IHL: deve ser pelo menos 5 (5 * 4 = 20 bytes).
    if (ip-&gt;ihl &lt; 5)
        return XDP_PASS;
    
    // Opcional: verificação completa do cabeçalho IP, considerando opções se presentes.
    if ((void *)ip &#43; (ip-&gt;ihl * 4) &gt; data_end)
        return XDP_PASS;

    // Extrai o IP de origem (localizado nos primeiros 20 bytes e sempre presente se ip-&gt;ihl &gt;= 5)
    __u32 src_ip = ip-&gt;saddr;

    // Consulta no mapa: se o IP de origem estiver presente, descarta o pacote.
    __u32 *blocked = bpf_map_lookup_elem(&amp;blocked_ips, &amp;src_ip);
    if (blocked)
        return XDP_DROP;

    return XDP_PASS;
}

// Obrigatório: licença do programa eBPF, necessária para que o kernel aceite o carregamento.
char _license[] SEC(&#34;license&#34;) = &#34;GPL&#34;;

```

### Detalhes Técnicos

1. **Verificações de Segurança e Integridade:**
    
    - **Cabeçalho Ethernet:**  
        O código garante que o cabeçalho Ethernet completo esteja presente, comparando o ponteiro para o fim do cabeçalho com `data_end`.
    - **Cabeçalho IP:**
        - É verificado se pelo menos 20 bytes (o tamanho mínimo do cabeçalho IP) estão disponíveis.
        - Verificação para o campo `ihl` (Internet Header Length), garantindo que seja maior ou igual a 5.
        - Se o IP possuir opções (quando `ihl &gt; 5`), o código valida que o cabeçalho completo (tamanho `ip-&gt;ihl * 4`) está dentro dos limites do buffer.


2. **Extração do IP de Origem:**
    
    - O IP de origem (`ip-&gt;saddr`) é extraído sem necessidade de cálculos adicionais, pois ele sempre se encontra nos primeiros 20 bytes do cabeçalho IP.

3. **Consulta no Mapa eBPF:**
    
    - O mapa `blocked_ips` é consultado utilizando o helper `bpf_map_lookup_elem`.
    - Se o ponteiro retornado for não nulo, isso indica que o IP está bloqueado e, consequentemente, o pacote é descartado com `XDP_DROP`.
    - Essa operação é extremamente rápida, pois o mapa foi definido para ter um acesso em tempo O(1) (tabela hash).

4. **Performance e Considerações de Otimização:**
    
    - **Execução no XDP:**  
        O programa é executado no XDP, logo na entrada dos pacotes, o que garante baixa latência e alta performance.
    - **Minimalismo:**  
        A lógica foi mantida mínima para reduzir o número de instruções e evitar qualquer latência adicional.
    - **Verificador eBPF:**  
        As verificações de limites são necessárias para passar pelo verificador do kernel, garantindo que o programa seja seguro e não acesse memória fora dos limites.
    - **Possíveis Micro-Otimizações:**  
        Em cenários mais avançados, técnicas como _branch prediction hints_ (por meio de macros como `__builtin_expect`) podem ser consideradas, mas normalmente o compilador BPF já otimiza o código de forma eficaz.

5. **Licenciamento:**
    
    - A inclusão da string de licença `&#34;GPL&#34;` é obrigatória para o carregamento do programa no kernel.


### Compilando

Utilize o clang para compilar para o formato eBPF:

```bash
clang -O2 -g -target bpf -c -Wall -Werror src/xdp-block-dos-0.1.0.c -o xdp-block-dos.o
```


## 3. Carregando na Interface de Rede

Para carregar o programa na interface, vamos usar  o comando  [ip](https://man7.org/linux/man-pages/man8/ip.8.html) do [iproute2](https://wiki.linuxfoundation.org/networking/iproute2)

```bash
sudo ip link set dev &lt;?&gt; xdp obj xdp-block-dos.o sec xdp
```

Para verificar se o programa foi carregado corretamente, utilize:

- note a anotação `prog/xdp id 87 name xdp_block_dos ...`

```bash
sudo ip -details link show dev enp1s0

2: enp1s0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 xdp qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:9a:8d:fd brd ff:ff:ff:ff:ff:ff promiscuity 0 allmulti 0 minmtu 68 maxmtu 65535 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 tso_max_size 65536 tso_max_segs 65535 gro_max_size 65536 gso_ipv4_max_size 65536 gro_ipv4_max_size 65536 parentbus virtio parentdev virtio1 
    prog/xdp id 87 name xdp_block_dos tag 20447cf6515f37e3 jited load_time 53777200546280 created_by_uid 0 btf_id 150 
```

Se necessário, para remover o programa:

```bash
sudo ip link set dev &lt;?&gt; xdp off
```



## 4. Script Bash para Monitoramento e Atualização do Mapa
Segue script em Bash que implementa uma abordagem híbrida para detectar um ataque DDoS: 

- Monitorando o número de conexões em [/proc/net/nf_conntrack](https://conntrack-tools.netfilter.org/manual.html#conntrack).

- E existindo um pico suspeito, analisa os logs do `nftables` para identificar IPs suspeitos devido a sua alta frequência.

- Para então atualizar o mapa eBPF (fixado em um caminho configurável) para bloquear os IPs suspeitos.

&gt;[!INFO] **RTFM** e código atualizado: [repo Git](https://github.com/0xttfx/monitor-dos)

---

```bash

#!/bin/bash

# Valores padrões
MAP_PIN_DEFAULT=&#34;/sys/fs/bpf/blocked_ips&#34;
THRESHOLD_CONN_DEFAULT=2000
LOG_THRESHOLD_DEFAULT=5
INTERVAL_DEFAULT=10
LOG_FILE_DEFAULT=&#34;/var/log/nftables.log&#34;

# Inicializa variáveis com os valores padrão
MAP_PIN=&#34;$MAP_PIN_DEFAULT&#34;
THRESHOLD_CONN=&#34;$THRESHOLD_CONN_DEFAULT&#34;
LOG_THRESHOLD=&#34;$LOG_THRESHOLD_DEFAULT&#34;
INTERVAL=&#34;$INTERVAL_DEFAULT&#34;
LOG_FILE=&#34;$LOG_FILE_DEFAULT&#34;

# Função para exibir a mensagem de ajuda
usage() {
    cat &lt;&lt; EOF
Uso: $(basename &#34;$0&#34;) [opções]

Opções:
  -m MAP_PIN         Caminho do mapa eBPF pinado (padrão: $MAP_PIN_DEFAULT)
  -c THRESHOLD_CONN  Número de conexões para acionar análise (padrão: $THRESHOLD_CONN_DEFAULT)
  -l LOG_THRESHOLD   Número mínimo de ocorrências de um IP nos logs para bloqueá-lo (padrão: $LOG_THRESHOLD_DEFAULT)
  -i INTERVAL        Intervalo de tempo para verificação em segundos (padrão: $INTERVAL_DEFAULT)
  -f LOG_FILE        Caminho para o arquivo de log do nftables (padrão: $LOG_FILE_DEFAULT)
  -h                 Exibe esta mensagem de ajuda e sai

Exemplo:
  sudo $(basename &#34;$0&#34;) -m /sys/fs/bpf/blocked_ips -c 1500 -l 10 -i 15 -f /var/log/nftables.log

EOF
}

# Processa as opções de linha de comando
while getopts &#34;m:c:l:i:f:h&#34; opt; do
    case &#34;$opt&#34; in
        m) MAP_PIN=&#34;$OPTARG&#34; ;;
        c) THRESHOLD_CONN=&#34;$OPTARG&#34; ;;
        l) LOG_THRESHOLD=&#34;$OPTARG&#34; ;;
        i) INTERVAL=&#34;$OPTARG&#34; ;;
        f) LOG_FILE=&#34;$OPTARG&#34; ;;
        h)
            usage
            exit 0
            ;;
        ?)
            usage
            exit 1
            ;;
    esac
done

echo &#34;Configurações utilizadas:&#34;
echo &#34;  MAP_PIN:         $MAP_PIN&#34;
echo &#34;  THRESHOLD_CONN:  $THRESHOLD_CONN&#34;
echo &#34;  LOG_THRESHOLD:   $LOG_THRESHOLD&#34;
echo &#34;  INTERVAL:        $INTERVAL&#34;
echo &#34;  LOG_FILE:        $LOG_FILE&#34;
echo

# Função para pinagem automática do mapa eBPF se ainda não estiver pinado.
auto_pin_map() {
    if [ ! -e &#34;$MAP_PIN&#34; ]; then
        echo &#34;O mapa eBPF não está pinado em $MAP_PIN. Tentando pinar automaticamente...&#34;
        # Procura o mapa &#34;blocked_ips&#34; usando bpftool.
        # O comando bpftool map show normalmente lista linhas como:
        # &#34;1: map 0 pinned /sys/fs/bpf/blocked_ips  ... blocked_ips&#34;
        # Aqui, usamos grep para encontrar a linha que contenha &#34;blocked_ips&#34;
        map_id=$(bpftool map show 2&gt;/dev/null | grep -m1 &#34;blocked_ips&#34; | awk &#39;{print $1}&#39; | tr -d &#39;:&#39;)
        if [ -n &#34;$map_id&#34; ]; then
            bpftool map pin id &#34;$map_id&#34; &#34;$MAP_PIN&#34;
            if [ $? -eq 0 ]; then
                echo &#34;Mapa &#39;blocked_ips&#39; (ID: $map_id) pinado com sucesso em $MAP_PIN.&#34;
            else
                echo &#34;Erro ao piná-lo. Verifique as permissões e a existência do mapa.&#34;
                exit 1
            fi
        else
            echo &#34;Erro: não foi possível encontrar o mapa &#39;blocked_ips&#39;.&#34;
            exit 1
        fi
    else
        echo &#34;Mapa eBPF já está pinado em $MAP_PIN.&#34;
    fi
}

# Executa a pinagem automática do mapa
auto_pin_map

# Função: Converte um IP (formato x.x.x.x) para valor hexadecimal (big-endian)
ip_to_hex() {
    local ip=&#34;$1&#34;
    IFS=&#39;.&#39; read -r a b c d &lt;&lt;&lt; &#34;$ip&#34;
    printf &#34;0x%02X%02X%02X%02X&#34; &#34;$a&#34; &#34;$b&#34; &#34;$c&#34; &#34;$d&#34;
}

# Função: Atualiza o mapa eBPF para bloquear um IP específico
block_ip() {
    local ip=&#34;$1&#34;
    local key_hex
    key_hex=$(ip_to_hex &#34;$ip&#34;)
    
    # Converte a string hexadecimal em bytes separados por espaços
    local key_bytes
    key_bytes=$(echo &#34;$key_hex&#34; | sed &#39;s/\(..\)/\1 /g&#39; | tr &#39;[:upper:]&#39; &#39;[:lower:]&#39;)
    
    echo &#34;Bloqueando IP suspeito: $ip (chave: $key_hex)&#34;
    bpftool map update pinned &#34;$MAP_PIN&#34; key hex $key_bytes value hex 01 00 00 00 2&gt;/dev/null
}

# Função: Retorna a contagem de conexões em /proc/net/nf_conntrack
get_conn_count() {
    if [ -f /proc/net/nf_conntrack ]; then
        wc -l &lt; /proc/net/nf_conntrack
    else
        echo &#34;0&#34;
    fi
}


# Função: Loop principal de monitoramento
main() {
    while true; do
        conn_count=$(get_conn_count)
        echo &#34;Conexões atuais: $conn_count&#34;

        if [ &#34;$conn_count&#34; -gt &#34;$THRESHOLD_CONN&#34; ]; then
            echo &#34;Alerta: Pico de conexões detectado ($conn_count &gt; $THRESHOLD_CONN).&#34;
            echo &#34;Analisando os logs do nftables para identificar IPs suspeitos...&#34;

            suspicious_ips=$(timeout &#34;$INTERVAL&#34; tail -n 1000 &#34;$LOG_FILE&#34; 2&gt;/dev/null | \
                grep &#34;SRC=&#34; | sed -n &#39;s/.*SRC=\([0-9.]\&#43;\).*/\1/p&#39; | sort | uniq -c | \
                awk -v thresh=&#34;$LOG_THRESHOLD&#34; &#39;$1 &gt;= thresh {print $2}&#39;)

            for ip in $suspicious_ips; do
                block_ip &#34;$ip&#34;
            done
        else
            echo &#34;Número de conexões dentro do esperado.&#34;
        fi

        sleep &#34;$INTERVAL&#34;
    done
}

# Executa o loop principal somente se o script estiver sendo chamado diretamente
if [[ &#34;${BASH_SOURCE[0]}&#34; == &#34;$0&#34; ]]; then
    main
fi
``` 

---

## Detalhes técnicos


### 1. Inicialização de Variáveis e Processamento de Opções

- **Definição de Valores Padrão:**  
  O script já define valores padrões para as variáveis:

  - `MAP_PIN_DEFAULT`           → `/sys/fs/bpf/blocked_ips`
  - `THRESHOLD_CONN_DEFAULT`    → 2000 (número de conexões para acionar a análise)
  - `LOG_THRESHOLD_DEFAULT`     → 5 (ocorrências para considerar um IP suspeito)
  - `INTERVAL_DEFAULT`          → 10 segundos (intervalo de verificação)
  - `LOG_FILE_DEFAULT`          → `/var/log/nftables.log`

- **Processamento com `getopts`:**  
  Utilizando `getopts`, o script permite sobrescrever os valores padrões:

  - `-m` para o caminho do mapa eBPF;
  - `-c` para o limite de conexões;
  - `-l` para o limite de ocorrências de um IP nos logs;
  - `-i` para o intervalo de tempo;
  - `-f` para o caminho do log do nftables;
  - `-h` exibi ajuda e sai ...o/

*Após processar os argumentos, os valores configurados são informados*

---

### 2. Função `auto_pin_map`
  
  Essa função garante que o mapa eBPF esteja “pinado” no caminho especificado.
  
- **Funcionamento:**  
  1. **Verificação da Existência:**  
     - Se o arquivo definido por `MAP_PIN` não existir, o script tenta localizar o mapa “blocked_ips” usando o comando:
       ```bash
       bpftool map show 2&gt;/dev/null | grep -m1 &#34;blocked_ips&#34; | awk &#39;{print $1}&#39; | tr -d &#39;:&#39;
       ```
     - Esse comando busca na saída do `bpftool map show` a linha que contenha “blocked_ips” e extrai o ID do mapa.
  2. **Pinagem do Mapa:**  
     - Se o ID for encontrado, o script utiliza:
       ```bash
       bpftool map pin id &#34;$map_id&#34; &#34;$MAP_PIN&#34;
       ```
       para fixar o mapa no caminho desejado.
  3. **Verificação de Erro:**  
     - Caso a pinagem falhe ou o mapa não seja encontrado, o script exibe uma mensagem de erro e encerra a execução.

---

### 3. Função `ip_to_hex`
 
  Converte o IP no formato decimal (ex.: `192.168.0.1`) para uma representação hexadecimal em big-endian, que é o formato esperado pela chave no mapa eBPF.
  
- **Como Funciona:**  
  - O comando `IFS=&#39;.&#39; read -r a b c d &lt;&lt;&lt; &#34;$ip&#34;` divide o IP nos seus quatro octetos.
  - O `printf &#34;0x%02X%02X%02X%02X&#34;` formata cada octeto em dois dígitos hexadecimais e os concatena.

---

### 4. Função `block_ip`

  Inserir no mapa eBPF (pinado em `MAP_PIN`) a chave correspondente a um IP suspeito.

- **Como Funciona:**  
  1. Converte o IP para hexadecimal utilizando `ip_to_hex`.
  2. Exibe uma mensagem indicando que o IP está sendo bloqueado, junto com sua chave hexadecimal.
  3. Utiliza o comando:
     ```bash
     bpftool map update pinned &#34;$MAP_PIN&#34; key hex $key_bytes value hex 01 00 00 00 2&gt;/dev/null
     ```
     para atualizar o mapa eBPF, inserindo a chave e definindo o valor (neste caso, `01` indica o bloqueio).

---

### 5. Função `get_conn_count`
  
  Conta o número de linhas em `/proc/net/nf_conntrack`, que corresponde ao número de conexões rastreadas pelo sistema.

- **Como Funciona:**  
  - Se o arquivo existe, o comando `wc -l` é utilizado para contar as linhas.
  - Caso o arquivo não exista, a função retorna “0”.

---

### 6.  Função `main`

- **Fluxo do Loop:**
  1. **Leitura do Número de Conexões:**  
     - A função `get_conn_count` é chamada para obter o número atual de conexões.
     - É exibida uma mensagem com o número de conexões.
  
  2. **Verificação de Pico de Conexões:**  
     - Se o número de conexões ultrapassa o limite definido por `THRESHOLD_CONN`, o script:
       - Exibe um alerta indicando que houve um pico.
       - Inicia a análise dos logs do nftables para identificar IPs suspeitos.
  
  3. **Análise dos Logs do nftables:**  
     - Utiliza uma cadeia de comandos para extrair os IPs:
       ```bash
       suspicious_ips=$(timeout &#34;$INTERVAL&#34; tail -n 1000 &#34;$LOG_FILE&#34; 2&gt;/dev/null | \
           grep &#34;SRC=&#34; | sed -n &#39;s/.*SRC=\([0-9.]\&#43;\).*/\1/p&#39; | sort | uniq -c | \
           awk -v thresh=&#34;$LOG_THRESHOLD&#34; &#39;$1 &gt;= thresh {print $2}&#39;)
       ```
       - **Passo a passo da cadeia:**
         - `tail -n 1000 &#34;$LOG_FILE&#34;`: Obtém as últimas 1000 linhas do log.
         - `timeout &#34;$INTERVAL&#34;`: Garante que a leitura não ultrapasse o intervalo definido.
         - `grep &#34;SRC=&#34;`: Filtra as linhas que contêm a string “SRC=”, que indica a presença de um endereço IP de origem.
         - `sed -n &#39;s/.*SRC=\([0-9.]\&#43;\).*/\1/p&#39;`: Extrai o endereço IP dessas linhas.
         - `sort | uniq -c`: Ordena os IPs e conta as ocorrências de cada um.
         - `awk -v thresh=&#34;$LOG_THRESHOLD&#34; &#39;$1 &gt;= thresh {print $2}&#39;`: Filtra apenas os IPs cuja contagem seja maior ou igual ao valor definido em `LOG_THRESHOLD`.
  
  4. **Bloqueio dos IPs Suspeitos:**  
     - Para cada IP identificado como suspeito na etapa anterior, a função `block_ip` é chamada para inserir o IP no mapa eBPF.
  
  5. **Intervalo de Espera:**  
     - Ao final de cada iteração, o script aguarda o número de segundos definido por `INTERVAL` antes de iniciar a próxima verificação.

- **Loop:**  
  O `while true; do ... done` mantém o monitoramento contínuo, permitindo uma resposta dinâmica a picos de tráfego.

---

### 8. Execução Condicional

- **Execução Direta:**  
  O trecho final:
  ```bash
  if [[ &#34;${BASH_SOURCE[0]}&#34; == &#34;$0&#34; ]]; then
      main
  fi
  ```
  Garante que o loop principal só seja iniciado se o script for executado diretamente 


## 5. Regras e o Log nftables
Geralmente, o nftables envia os logs para o kernel, e com o rsyslog podemos filtrar essas mensagens e gravá-las em um arquivo dedicado.

### Regras nftables

Segue um exemplo de regras específicas para a solução e que gerará um Log com prefixo `DDoS_ALERT: SRC=` definido:

```nft
table inet ddos_filter {
    chain input {
        type filter hook input priority 0; policy accept;
        
        ip protocol tcp ct state new limit rate 10/second burst 20 packets \
            log prefix &#34;DDoS_ALERT: SRC=&#34; flags all

        ip protocol udp ct state new limit rate 10/second burst 20 packets \
            log prefix &#34;DDoS_ALERT: SRC=&#34; flags all

        ip protocol icmp ct state new limit rate 5/second burst 10 packets \
            log prefix &#34;DDoS_ALERT: SRC=&#34; flags all
    }
}
```

- Com essa configuração, os eventos de log contendo `&#34;DDoS_ALERT: SRC=&#34;` serão redirecionados para o arquivo `/var/log/nftables.log`, permitindo que o script extraia os IPs suspeitos...


### Configurar o Log

1. **Garanta que o serviço esteja rodando**  
    No Fedora, ambiente de teste usado, o `rsyslog` geralmente vem instalado, mas certifique-se de que ele está em execução:
    
    ```bash
    sudo systemctl status rsyslog
    ```
    
2. **Filtrando os Logs**  
    Crie um arquivo de configuração para o rsyslog que irá filtrar nosso log
     
   ```bash
   touch /etc/rsyslog.d/nftables.conf
   ```
	
	1. E adicione uma regra para filtrar as mensagens que contenham o prefixo  `&#34;DDoS_ALERT: SRC=&#34;`.
		```rsyslog
	    # Filtra mensagens que contenham &#34;DDoS_ALERT:&#34; e as grava em /var/log/nftables.log
	    :msg, contains, &#34;DDoS_ALERT:&#34; -/var/log/nftables.log
	    &amp; ~
	    ```
		- A primeira linha diz ao rsyslog: se a mensagem contiver o texto `&#34;DDoS_ALERT:&#34;`, envie-a para o arquivo `/var/log/nftables.log`. O hífen (`-`) antes do caminho indica gravação assíncrona, reduzindo o overhead.
		- A linha `&amp; ~` impede que essas mensagens sejam processadas por outras regras, evitando duplicidade.
	
	2. **Reinicie o serviço**
	 ```bash
     sudo systemctl restart rsyslog
     ```
    
3. **(Opcional) Configure o systemd-journald para encaminhar os logs ao rsyslog:**  
    No Fedora, o `journald` já encaminha as mensagens para o `rsyslog` por padrão, mas se necessário, verifique o arquivo `/etc/systemd/journald.conf` e certifique-se de que a linha abaixo esteja configurada:
    
    ```bash
    ForwardToSyslog=yes
    ```
    
4. **Verifique se os logs estão sendo gravados:**  
    Após aplicar a configuração e reiniciar o rsyslog, você pode testar sua configuração gerando uma mensagem de log (por exemplo, criando um evento com nftables) e verificando o conteúdo do arquivo:
    
    ```bash
    sudo tail -f /var/log/nftables.log
    ```


## 6. Testes de Validação

Para validar a solução. você pode seguir uma série de testes que abrangem cada componente e a integração completa. Abaixo, segue um passo a passo detalhado com sugestões de ferramentas e métodos:


## 1. Preparação do Ambiente
- **Requisitos:**
    
    - 02 VMs Linux Fedora 41.
    - Ferramentas instaladas:
	    - clang/llvm, 
	    - bpftool, 
	    - nftables, 
	    - rsyslog, 
	    - tcpdump, 
	    - hping3 ou scapy.
    - SUDO para realizar os passos que exijam privilégio root.


## 2. Testando Individualmente Cada Componente

### 2.1. Programa XDP/eBPF
1. **Diretório de teste**

```bash
$ mkdir /tmp/src &amp;&amp; cd /tmp/src
```

1. **Compilação e Carregamento:**
    
    - Compile o programa:
	    - Clone o repositório
	
        ```bash
        $ git clone https://github.com/0xttfx/xdp-block-dos.git
        ```
    - Compile o código

	    ```bash
	    $ make all
		
		clang -O2 -g -target bpf -c -Wall -Werror src/xdp-block-dos-0.1.0.c -o xdp-block-dos.o
	    ```        
    - Carregue o programa na interface de rede (supondo `eth0`):
        
        ```bash
        sudo ip link set dev eth0 xdp obj xdp_ddos_blocker.o sec xdp
        ```
        
3. **Verificação:**
    
    - Verifique se o programa está carregado:
        
    ```bash
    $ sudo ip -details link show dev enp1s0
		 
	2: enp1s0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 xdp qdisc fq_codel state UP mode DEFAULT group default qlen 1000
		    link/ether 52:54:00:9a:8d:fd brd ff:ff:ff:ff:ff:ff promiscuity 0 allmulti 0 minmtu 68 maxmtu 65535 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 tso_max_size 65536 tso_max_segs 65535 gro_max_size 65536 gso_ipv4_max_size 65536 gro_ipv4_max_size 65536 parentbus virtio parentdev virtio1 
	prog/xdp id 92 name xdp_block_dos tag 20447cf6515f37e3 jited load_time 2787963305254 created_by_uid 0 btf_id 140
    ```
    - Veja se o mapa foi criado e colete o ID

	 ```bash
	 $ sudo bpftool map show 2&gt;/dev/null | grep -m1 &#34;blocked_ips&#34; | awk &#39;{print $1}&#39; | tr -d &#39;:&#39;
	 22
	 ```
    - Pine o arquivo `map`
    
    ```bash
    $ sudo bpftool map pin id &#34;22&#34; &#34;/sys/fs/bpf/blocked_ips&#34;
	```
	- Use o `bpftool` para fazer um dump do `map` eBPF
	
	```bash
	$ sudo bpftool map dump pinned /sys/fs/bpf/blocked_ips
	[]
    ```

4. **Teste Funcional:**
    
    - Gere tráfego a partir de uma máquina ou usando ferramentas como `hping3` de um IP que não esteja bloqueado
	    - Verifique que os pacotes seguem normalmente (XDP_PASS).
		    - Em seguida, adicione manualmente o IP ao mapa (usando o bpftool)
			    - E verifique se os pacotes são descartados (XDP_DROP).
			
		    1. Converta o IP da VM que está disparando o tráfego, para hexadecima
	    
		    ```bash
		    $  ip=&#34;192.168.0.115&#34;; IFS=&#39;.&#39;; read -r a b c d &lt;&lt;&lt; &#34;$ip&#34;; printf &#34;%02X%02X%02X%02X\n&#34; &#34;$a&#34; &#34;$b&#34; &#34;$c&#34; &#34;$d&#34; | sed &#39;s/\(..\)/\1 /g&#39; | tr &#39;[:upper:]&#39; &#39;[:lower:]&#39;
			c0 a8 00 73 
		    ```
		    1. Agora insira o ip no `map` eBPF
	    
		    ```bash
		    $ sudo bpftool map update pinned &#39;/sys/fs/bpf/blocked_ips&#39; key hex C0 A8 00 73 value hex 01 00 00 00
		    ```
			1. E faça um dump do `map`para verificar se a insserçã́o ocorreu
		
			```bash
			$ sudo bpftool map dump pinned /sys/fs/bpf/blocked_ips
			[{
		        &#34;key&#34;: 1929423040,
		        &#34;value&#34;: 1
			    }
			]
			```
		     1.  E para quando for preciso. Esse é o comando para remover a entrada que criamos
			 ```bash
			 sudo bpftool map delete pinned &#39;/sys/fs/bpf/blocked_ips&#39; key hex c0 a8 00 73
			 ```

5. **Evidências**

&lt;style&gt;
  .video-container {
    max-width: 100%;
    height: auto;
  }
&lt;/style&gt;

&lt;video class=&#34;video-container&#34; controls&gt;
  &lt;source src=&#34;/img/XDPeBPFDoS/evidencia-1.mp4&#34; type=&#34;video/mp4&#34;&gt;
  Seu navegador não suporta o elemento de vídeo.
&lt;/video&gt;


### 2.2. Configuração do nftables e Logs
1. **Diretório de teste**
	1. Acesse ou crie o diretório...

```bash
$ mkdir /tmp/src &amp;&amp; cd /tmp/src
```

2. **Regras nftables**
	1. Crie o arquivos `dos.nft`
	```nft
	cat &lt;&lt;Uai&gt; /tmp/src/dos.nft
	table inet dos_filter {
		chain input {
			type filter hook input priority 0; policy accept;
			
			ip protocol tcp ct state new limit rate 10/second burst 20 packets \
				log prefix &#34;DDoS_ALERT: SRC=&#34; flags all
			   
			ip protocol udp ct state new limit rate 10/second burst 20 packets \
				log prefix &#34;DDoS_ALERT: SRC=&#34; flags all
		}
	}
	Uai
	```

	2. Carregue as regras
	
	```bash
	sudo nft -f ddos.nft
	```
	
	3. Verifique as regras da table criada

	```bash
	$ sudo nft list table inet dos_filter
	
	table inet dos_filter {
		chain input {
			type filter hook input priority filter; policy accept;
			ip protocol tcp ct state new limit rate 10/second burst 20 packets log prefix &#34;DDoS_ALERT: SRC=&#34; flags all
			ip protocol udp ct state new limit rate 10/second burst 20 packets log prefix &#34;DDoS_ALERT: SRC=&#34; flags all
		}
	}
	```


3. **Rsyslog:**

	1. Crie o arquivo de configuração em `/etc/rsyslog.d/nftables.conf` 
	
	```bash
	sudo su -c &#39;cat &lt;&lt;Uai&gt; /etc/rsyslog.d/dos_nftables.conf
	# Filtra mensagens que contenham &#34;DDoS_ALERT:&#34; e as grava em /var/log/dos_nftables.log
	:msg, contains, &#34;DDoS_ALERT:&#34; -/var/log/dos_nftables.log
	&amp; stop
	Uai&#39;
	```
	2. Recarregue as confs para o systemd
	
	```bash
	sudo systemctl daemon-reload
	```
	1. Reinicie o rsyslog:
	
	```bash
	sudo systemctl restart rsyslog
    ```
    1. Visualize os logs para confirmar que as entradas com prefixo &#34;DDoS_ALERT: SRC=&#34; estão sendo gravadas em `/var/log/dos_nftables.log`:
    
    ```bash
    sudo tail -f /var/log/dos_nftables.log
    ```
        
2. **Geração de Logs:**
    - Para testar, você pode simular novos fluxos TCP, UDP com`hping3`.

	1. Para ver o comportamento para protocolo UDP
		1. na VM 1: crie um servidor UDP
		```bash
		cat &lt;&lt;Uai&gt; /tmp/src/srv-udp.py
		import socket
		
		UDP_IP = &#34;0.0.0.0&#34;
		UDP_PORT = 9999
		
		sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
		sock.bind((UDP_IP, UDP_PORT))
		print(&#34;Servidor UDP escutando na porta&#34;, UDP_PORT)
		
		while True:
		    data, addr = sock.recvfrom(1024)  # Buffer de 1024 bytes
		    print(&#34;Recebido:&#34;, data.decode(), &#34;de&#34;, addr)
		Uai
		```

		 2. e execute o server
		```bash
		sudo python3 srv-udp.py
		```
		 
		 1. Na VM2, dispare contra a porta 9999 UDP
		```bash
		sudo hping3 -2 -p 9999 -i u10000 192.168.0.114
		```

		 1. Na VM1, observe os logs em
			 1. nf_conntrack
			 
			```bash
			sudo watch -n1 tail -f /proc/net/nf_conntrack
			```
			 1. dos_nftables.log
		    ```bash
		    sudo tail -f /var/log/dos_nftables.log
		    ```




&lt;script src=&#34;https://giscus.app/client.js&#34;
        data-repo=&#34;0xttfx/0xttfx.github.io&#34;
        data-repo-id=&#34;R_kgDOK3wAHw&#34;
        data-category=&#34;BlogPostComments&#34;
        data-category-id=&#34;DIC_kwDOK3wAH84Cnmtb&#34;
        data-mapping=&#34;pathname&#34;
        data-strict=&#34;1&#34;
        data-reactions-enabled=&#34;1&#34;
        data-emit-metadata=&#34;0&#34;
        data-input-position=&#34;top&#34;
        data-theme=&#34;preferred_color_scheme&#34;
        data-lang=&#34;en&#34;
        data-loading=&#34;lazy&#34;
        crossorigin=&#34;anonymous&#34;
        async&gt;
&lt;/script&gt;


---

> Author: [Faioli a.k.a 0xttfx](https://github.com/0xttfx)  
> URL: http://0xttfx.tcpip.net.br/posts/xdpebpf-block-ddos/  

