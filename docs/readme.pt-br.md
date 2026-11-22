# AVISO: CUIDADO COM GOLPES (SCAMMERS)

O zapret é um software livre e de código aberto, sob a [licença MIT](./LICENSE.txt).
Se alguém exigir que você baixe o zapret apenas do site/canal dela, que apague links, vídeos e arquivos, alegando direitos autorais para justificar essas exigências, essa pessoa está violando a [licença](./LICENSE.txt) — e é um golpista.
Isso não impede [doações voluntárias](#doações).

# zapret2

Esta versão do zapret não é mais desenvolvida e está em modo EOL (End-Of-Life / fim de vida). Não haverá novas funcionalidades, apenas correções de bugs. Pull requests com novas funcionalidades não serão aceitos.

[Versão atual — zapret 2](https://github.com/bol-van/zapret2)

# README multilíngue / Multilanguage README
___
[![en](https://img.shields.io/badge/lang-en-red.svg)](./readme.en.md)
[![pt-BR](https://img.shields.io/badge/lang-pt--BR-blue.svg)](./readme.pt-br.md)
[![ru](https://img.shields.io/badge/lang-ru-green.svg)](./readme.md)

***

- [Para que serve](#para-que-serve)
- [Como funciona](#como-funciona)
- [Como colocar isso em prática no sistema Linux](#como-colocar-isso-em-prática-no-sistema-linux)
- [Quando isso não funcionará](#quando-isso-não-funcionará)
- [nfqws](#nfqws)
  - [Ataque de dessincronização do DPI](#ataque-de-dessincronização-do-dpi)
  - [Fakes](#fakes)
  - [Modificações de fakes](#modificações-de-fakes)
  - [Segmentação TCP](#segmentação-tcp)
  - [Sobreposição de números de sequência](#sobreposição-de-números-de-sequência)
  - [Atribuição de IP_ID](#atribuição-de-ip_id)
  - [Modos específicos de ipv6](#modos-específicos-de-ipv6)
  - [Modificação do original](#modificação-do-original)
  - [Duplicatas](#duplicatas)
  - [Reação à resposta do servidor](#reação-à-resposta-do-servidor)
  - [Modo SYNDATA](#modo-syndata)
  - [Combinações de dessincronização do DPI](#combinações-de-dessincronização-do-dpi)
  - [Cache de IP](#cache-de-ip)
  - [CONNTRACK](#conntrack)
  - [Remontagem (Reassemble)](#remontagem-reassemble)
  - [Suporte a UDP](#suporte-a-udp)
  - [Fragmentação de IP](#fragmentação-de-ip)
  - [Múltiplas estratégias](#múltiplas-estratégias)
  - [Filtragem por WIFI](#filtragem-por-wifi)
  - [Máquinas virtuais](#máquinas-virtuais)
  - [IPTABLES para nfqws](#iptables-para-nfqws)
  - [NFTABLES para nfqws](#nftables-para-nfqws)
  - [Flow offloading (descarregamento de fluxo)](#flow-offloading-descarregamento-de-fluxo)
  - [Engano no lado do servidor](#engano-no-lado-do-servidor)
- [tpws](#tpws)
  - [Segmentação TCP no tpws](#segmentação-tcp-no-tpws)
  - [TLSREC](#tlsrec)
  - [MSS](#mss)
  - [Outras opções de modificação](#outras-opções-de-modificação)
  - [Opções suplementares](#opções-suplementares)
  - [Múltiplas estratégias](#múltiplas-estratégias-1)
  - [IPTABLES para tpws](#iptables-para-tpws)
  - [NFTABLES para tpws](#nftables-para-tpws)
- [Formas de obter uma lista de IPs bloqueados](#formas-de-obter-uma-lista-de-ips-bloqueados)
- [Filtragem por nome de domínio](#filtragem-por-nome-de-domínio)
- [Modo autohostlist](#modo-autohostlist)
- [Escolhendo parâmetros](#escolhendo-parâmetros)
- [Integrando ao sistema de controle de firewall ou ao seu sistema de inicialização](#integrando-ao-sistema-de-controle-de-firewall-ou-ao-seu-sistema-de-inicialização)
- [Instalação](#instalação)
  - [Verificando o ISP](#verificando-o-isp)
  - [sistema Linux desktop](#sistema-linux-desktop)
  - [OpenWRT](#openwrt)
  - [Android](#android)
  - [FreeBSD, OpenBSD, MacOS](#freebsd-openbsd-macos)
  - [Windows (WSL)](#windows-wsl)
  - [Outros dispositivos](#outros-dispositivos)
- [Doações](#doações)

## Para que serve

Uma ferramenta independente (sem servidores de terceiros) para burlar o DPI (Deep Packet Inspection).
Pode permitir contornar o bloqueio de sites http(s) ou a limitação de velocidade (throttling), e resistir à detecção de assinaturas de protocolos TCP/UDP.

O projeto é voltado principalmente ao público russo.
Algumas funcionalidades são específicas da realidade russa (como obter a lista de sites bloqueados pela Roskomnadzor), mas a maioria é de uso geral.

O foco principal é o OpenWRT, mas também há suporte para Linux tradicional, FreeBSD, OpenBSD, Windows e, parcialmente, MacOS.

A maioria das funcionalidades também é suportada no Windows.

## Como funciona

No caso mais simples, você lida com um DPI passivo. Um DPI passivo consegue ler o tráfego que passa por ele e injetar seus próprios pacotes, mas não consegue descartar pacotes.

Se a requisição for proibida, o DPI passivo injeta seu próprio pacote RST e, opcionalmente, um pacote de redirecionamento http.

Se os pacotes falsos do DPI são enviados apenas ao cliente, você pode usar comandos iptables para descartá-los, desde que escreva regras de filtro corretas. Isso exige análise manual e detalhada do tráfego e ajustes específicos para cada ISP.

É assim que contornamos as consequências do gatilho de bloqueio.

Se o DPI passivo envia um pacote RST também ao servidor, não há nada que você possa fazer. Sua tarefa é impedir que o gatilho de bloqueio dispare. Apenas iptables não resolve.
Este projeto busca **prevenir o bloqueio**, e não eliminar suas consequências.

Para isso, envie algo que o DPI não espera e que quebra o algoritmo dele de reconhecer requisições e bloqueá-las.

Alguns DPIs não reconhecem uma requisição http se ela for dividida em segmentos TCP.
Por exemplo, uma requisição do tipo `GET / HTTP / 1.1 \ r \ nHost: kinozal.tv ......` é enviada em 2 partes: primeiro vai `GET`, depois `/ HTTP / 1.1 \ r \ nHost: kinozal.tv .....`.

Outros DPIs se confundem quando o cabeçalho `Host:` é escrito com outra capitalização: por exemplo, `host:`.

Às vezes funciona adicionar um espaço extra depois do método: `GET /` => `GET  /`
ou adicionar um ponto no final do nome do host: `Host: kinozal.tv.`.

Também existe uma mágica mais avançada para burlar o DPI no nível de pacote.

## Como colocar isso em prática no sistema Linux

Em resumo, as opções podem ser classificadas de acordo com o seguinte esquema:

1. DPI passivo que não envia RST ao servidor. Comandos iptables ajustados para o ISP podem ajudar.
Essa opção está fora do escopo do projeto. Se você não deixar o gatilho de bloqueio disparar, não terá que lidar com as consequências.
2. Modificação da conexão TCP no nível de fluxo. Implementada por meio de um proxy ou proxy transparente.
3. Modificação da conexão TCP no nível de pacote. Implementada por meio do manipulador NFQUEUE e raw sockets.

Para as opções 2 e 3, foram implementados os programas **tpws** e **nfqws**, respectivamente.
Você precisa executá-los com os parâmetros necessários e redirecionar determinado tráfego com iptables ou nftables.

## Quando isso não funcionará

* Se o servidor DNS retornar respostas falsas. O ISP pode retornar IPs falsos ou não retornar nada
quando domínios bloqueados são consultados. Se for esse o caso, troque o DNS por um público, como 8.8.8.8 ou 1.1.1.1. Às vezes o ISP sequestra consultas a qualquer servidor DNS. Dnscrypt ou dns-over-tls ajudam.
* Se o bloqueio for feito por IP.
* Se a conexão passa por um filtro capaz de reconstruir uma conexão TCP e que segue todos os padrões. Por exemplo, somos roteados para um squid. A conexão passa pela pilha TCP/IP completa do sistema operacional. Este projeto visa apenas o DPI, não a pilha completa do SO nem aplicações de servidor.

## nfqws

Este programa é um modificador de pacotes e um manipulador da fila NFQUEUE.
Para sistemas BSD existe o dvtws. Ele é compilado do mesmo código-fonte e tem praticamente os mesmos parâmetros (veja [bsd.en.md](./bsd.en.md)).
O nfqws aceita os seguintes parâmetros:

```
 @<config_file>                                            ; lê arquivo de opções. deve ser o único argumento. outras opções são ignoradas.

 --debug=0|1
 --dry-run                                                 ; valida os parâmetros e sai com código 0 se bem-sucedido
 --version                                                 ; imprime a versão e sai
 --comment                                                 ; qualquer texto (ignorado)
 --qnum=<nfqueue_number>
 --daemon                                                  ; executa como daemon
 --pidfile=<filename>                                      ; grava o pid em arquivo
 --user=<username>                                         ; remove privilégios de root
 --uid=uid[:gid1,gid2,...]                                 ; remove privilégios de root
 --bind-fix4                                               ; aplica correção de seleção de interface de saída para pacotes ipv4 gerados
 --bind-fix6                                               ; aplica correção de seleção de interface de saída para pacotes ipv6 gerados
 --wsize=<window_size>[:<scale_factor>]                    ; define tamanho da janela. 0 = não modificar. OBSOLETO !
 --wssize=<window_size>[:<scale_factor>]                   ; define tamanho da janela para o servidor. 0 = não modificar. scale_factor padrão = 0.
 --wssize-cutoff=[n|d|s]N                                  ; aplica wssize do servidor apenas a números de pacote (n, padrão), números de pacote de dados (d), sequência relativa (s) menores que N
 --wssize-forced-cutoff=0|1                                ; 1(padrão)=interrompe automaticamente o wssize em protocolo conhecido
 --ctrack-timeouts=S:E:F[:U]                               ; timeouts internos de conntrack para estágios TCP SYN, ESTABLISHED, FIN e UDP. padrão 60:300:60:60
 --ctrack-disable=[0|1]                                    ; 1 ou sem argumento desabilita o conntrack
 --ipcache-lifetime=<int>                                  ; tempo em segundos para manter em cache a contagem de saltos e o nome de domínio (padrão 7200). 0 = sem expiração
 --ipcache-hostname=[0|1]                                  ; 1 ou sem argumento habilita cache ip->hostname
 --hostcase                                                ; altera Host: => host:
 --hostspell                                               ; grafia exata do cabeçalho "Host". deve ter 4 caracteres. padrão é "host"
 --hostnospace                                             ; remove o espaço depois de Host: e o adiciona a User-Agent: para preservar o tamanho do pacote
 --domcase                                                 ; mistura capitalização do domínio : Host: TeSt.cOm
 --methodeol                                               ; adiciona '\n' antes do método e remove o espaço depois de Host:
 --synack-split=[syn|synack|acksyn]                        ; realiza handshake TCP dividido : envia apenas SYN, SYN+ACK ou ACK+SYN
 --orig-ttl=<int>                                          ; define TTL para pacotes originais
 --orig-ttl6=<int>                                         ; define hop limit ipv6 para pacotes originais. por padrão usa o valor de ttl
 --orig-autottl=[<delta>[:<min>[-<max>]]|-]                ; modo ttl automático para ipv4 e ipv6. padrão: +5:3-64. "0:0-0" ou "-" desabilita o autottl.
 --orig-autottl6=[<delta>[:<min>[-<max>]]|-]               ; substitui --orig-autottl apenas para ipv6
 --orig-tcp-flags-set=<int|0xHEX|flaglist>                 ; define estas flags tcp (flags |= value). valor pode ser int, hex ou lista separada por vírgula : FIN,SYN,RST,PSH,ACK,URG,ECE,CWR,AE,R1,R2,R3
 --orig-tcp-flags-unset=<int|0xHEX|flaglist>               ; remove estas flags tcp (flags &= ~value)
 --orig-mod-start=[n|d|s]N                                 ; aplica mod de TTL original a números de pacote (n, padrão), números de pacote de dados (d), sequência relativa (s) maiores ou iguais a N
 --orig-mod-cutoff=[n|d|s]N                                ; aplica mod de TTL original a números de pacote (n, padrão), números de pacote de dados (d), sequência relativa (s) menores que N
 --dup=<int>                                               ; duplica pacotes originais. envia N duplicatas antes do original.
 --dup-replace=[0|1]                                       ; 1 ou sem argumento significa não enviar o original, apenas as duplicatas
 --dup-ttl=<int>                                           ; define TTL para duplicatas
 --dup-ttl6=<int>                                          ; define hop limit ipv6 para duplicatas. por padrão usa o valor de ttl
 --dup-autottl=[<delta>[:<min>[-<max>]]|-]                 ; modo ttl automático para ipv4 e ipv6. padrão: -1:3-64. "0:0-0" ou "-" desabilita o autottl.
 --dup-autottl6=[<delta>[:<min>[-<max>]]|-]                ; substitui --dup-autottl apenas para ipv6
 --dup-tcp-flags-set=<int|0xHEX|flaglist>                  ; define estas flags tcp (flags |= value). valor pode ser int, hex ou lista separada por vírgula : FIN,SYN,RST,PSH,ACK,URG,ECE,CWR,AE,R1,R2,R3
 --dup-tcp-flags-unset=<int|0xHEX|flaglist>                ; remove estas flags tcp (flags &= ~value)
 --dup-fooling=<mode>[,<mode>]                             ; pode usar múltiplos valores separados por vírgula. modos : none md5sig badseq badsum datanoack hopbyhop hopbyhop2
 --dup-ts-increment=<int|0xHEX>                            ; incremento com sinal de TSval para engano de ts em duplicata. padrão -600000
 --dup-badseq-increment=<int|0xHEX>                        ; incremento com sinal de seq para engano de badseq em duplicata. padrão -10000
 --dup-badack-increment=<int|0xHEX>                        ; incremento com sinal de ackseq para engano de badseq em duplicata. padrão -66000
 --dup-ip-id=same|zero|seq|rnd                             ; modo de ip_id ipv4 para pacotes duplicados
 --dup-start=[n|d|s]N                                      ; aplica dup a números de pacote (n, padrão), números de pacote de dados (d), sequência relativa (s) maiores ou iguais a N
 --dup-cutoff=[n|d|s]N                                     ; aplica dup a números de pacote (n, padrão), números de pacote de dados (d), sequência relativa (s) menores que N
 --ip-id=zero|seq|seqgroup|rnd                             ; esquema de atribuição de ip_id ipv4
 --dpi-desync=[<mode0>,]<mode>[,<mode2>]                   ; tenta dessincronizar o estado do DPI. modos : synack fake fakeknown rst rstack hopbyhop destopt ipfrag1 multisplit multidisorder fakedsplit hostfakesplit fakeddisorder ipfrag2 udplen tamper
 --dpi-desync-fwmark=<int|0xHEX>                           ; substitui fwmark do pacote de dessincronização. padrão = 0x40000000 (1073741824)
 --dpi-desync-ttl=<int>                                    ; define ttl para o pacote de dessincronização
 --dpi-desync-ttl6=<int>                                   ; define hop limit ipv6 para o pacote de dessincronização. por padrão usa o valor de ttl.
 --dpi-desync-autottl=[<delta>[:<min>[-<max>]]|-]          ; modo ttl automático para ipv4 e ipv6. padrão: -1:3-20. "0:0-0" ou "-" desabilita o autottl.
 --dpi-desync-autottl6=[<delta>[:<min>[-<max>]]|-]         ; substitui --dpi-desync-autottl apenas para ipv6
 --dpi-desync-tcp-flags-set=<int|0xHEX|flaglist>           ; define estas flags tcp (flags |= value). valor pode ser int, hex ou lista separada por vírgula : FIN,SYN,RST,PSH,ACK,URG,ECE,CWR,AE,R1,R2,R3
 --dpi-desync-tcp-flags-unset=<int|0xHEX|flaglist>         ; remove estas flags tcp (flags &= ~value)
 --dpi-desync-fooling=<mode>[,<mode>]                      ; pode usar múltiplos valores separados por vírgula. modos : none md5sig ts badseq badsum datanoack hopbyhop hopbyhop2
 --dpi-desync-repeats=<N>                                  ; envia cada pacote de dessincronização N vezes
 --dpi-desync-skip-nosni=0|1                               ; 1(padrão)=não agir em ClientHello sem SNI (ESNI ?)
 --dpi-desync-split-pos=N|-N|marker+N|marker-N             ; lista de posições de divisão separadas por vírgula
                                                            ; marcadores: method,host,endhost,sld,endsld,midsld,sniext
                                                            ; a lista completa é usada apenas por multisplit e multidisorder
                                                            ; fakedsplit/fakeddisorder usam o primeiro parâmetro compatível com l7, se presente, senão o primeiro valor absoluto
 --dpi-desync-split-seqovl=N|-N|marker+N|marker-N          ; usa sobreposição de sequência antes do primeiro segmento original enviado
 --dpi-desync-split-seqovl-pattern=[+ofs]@<filename>|0xHEX ; padrão para a parte falsa da sobreposição
 --dpi-desync-fakedsplit-pattern=[+ofs]@<filename>|0xHEX   ; padrão falso para fakedsplit/fakeddisorder
 --dpi-desync-fakedsplit-mod=mod[,mod]                     ; mods podem ser none,altorder=0|1|2|3 + 0|8|16
 --dpi-desync-hostfakesplit-midhost=marker+N|marker-N      ; divide adicionalmente o hostname real no marcador especificado. deve estar entre host..endhost ou não será dividido.
 --dpi-desync-hostfakesplit-mod=mod[,mod]                  ; pode ser none, host=<hostname>, altorder=0|1
 --dpi-desync-ipfrag-pos-tcp=<8..9216>                     ; posição de fragmentação ip a partir do cabeçalho de transporte. múltiplo de 8, padrão 32.
 --dpi-desync-ipfrag-pos-udp=<8..9216>                     ; posição de fragmentação ip a partir do cabeçalho de transporte. múltiplo de 8, padrão 8.
 --dpi-desync-ts-increment=<int|0xHEX>                     ; incremento com sinal de TSval para engano de ts. padrão -600000
 --dpi-desync-badseq-increment=<int|0xHEX>                 ; incremento com sinal de seq para engano de badseq. padrão -10000
 --dpi-desync-badack-increment=<int|0xHEX>                 ; incremento com sinal de ackseq para engano de badseq. padrão -66000
 --dpi-desync-any-protocol=0|1                             ; 0(padrão)=dessincroniza apenas http e tls  1=dessincroniza qualquer pacote com dados
 --dpi-desync-fake-tcp-mod=mod[,mod]                       ; lista de mods de fake tcp separados por vírgula. mods disponíveis : none,seq
 --dpi-desync-fake-http=[+ofs]@<filename>|0xHEX            ; arquivo contendo uma requisição http falsa
 --dpi-desync-fake-tls=[+ofs]@<filename>|0xHEX|![+offset]  ; arquivo contendo um TLS ClientHello falso (para https). '!' = fake padrão
 --dpi-desync-fake-tls-mod=mod[,mod]                       ; lista de mods de fake TLS separados por vírgula. mods disponíveis : none,rnd,rndsni,sni=<sni>,dupsid,padencap
 --dpi-desync-fake-unknown=[+ofs]@<filename>|0xHEX         ; arquivo contendo payload falso de protocolo desconhecido
 --dpi-desync-fake-syndata=[+ofs]@<filename>|0xHEX         ; arquivo contendo payload de dados SYN
 --dpi-desync-fake-quic=[+ofs]@<filename>|0xHEX            ; arquivo contendo um QUIC Initial falso
 --dpi-desync-fake-wireguard=[+ofs]@<filename>|0xHEX       ; arquivo contendo um handshake de inicialização wireguard falso
 --dpi-desync-fake-dht=[+ofs]@<filename>|0xHEX             ; arquivo contendo DHT falso (d1..e)
 --dpi-desync-fake-discord=[+ofs]@<filename>|0xHEX         ; arquivo contendo pacote falso de início de chamada de voz do Discord (IP Discovery)
 --dpi-desync-fake-stun=[+ofs]@<filename>|0xHEX            ; arquivo contendo mensagem STUN falsa
 --dpi-desync-fake-unknown-udp=[+ofs]@<filename>|0xHEX     ; arquivo contendo payload falso de protocolo udp desconhecido
 --dpi-desync-udplen-increment=<int>                       ; aumenta ou diminui o tamanho do pacote udp em N bytes (padrão 2). valores negativos diminuem o tamanho.
 --dpi-desync-udplen-pattern=[+ofs]@<filename>|0xHEX       ; padrão de preenchimento do final udp
 --dpi-desync-start=[n|d|s]N                               ; aplica dpi desync apenas a números de pacote (n, padrão), números de pacote de dados (d), sequência relativa (s) maiores ou iguais a N
 --dpi-desync-cutoff=[n|d|s]N                              ; aplica dpi desync apenas a números de pacote (n, padrão), números de pacote de dados (d), sequência relativa (s) menores que N
 --hostlist=<filename>                                     ; aplica dpi desync apenas aos hosts listados (um host por linha, subdomínios aplicados automaticamente se não prefixados com `^`, gzip suportado, múltiplas listas permitidas)
 --hostlist-domains=<domain_list>                          ; lista fixa de domínios separados por vírgula
 --hostlist-exclude=<filename>                             ; não aplicar dpi desync aos hosts listados (um host por linha, subdomínios aplicados automaticamente se não prefixados com `^`, gzip suportado, múltiplas listas permitidas)
 --hostlist-exclude-domains=<domain_list>                  ; lista fixa de domínios separados por vírgula
 --hostlist-auto=<filename>                                ; detecta bloqueios de DPI e constrói a lista de hosts automaticamente
 --hostlist-auto-fail-threshold=<int>                      ; quantas tentativas falhas adicionam o hostname à lista automática (padrão : 3)
 --hostlist-auto-fail-time=<int>                           ; todas as falhas devem ocorrer nesse intervalo de segundos (padrão : 60)
 --hostlist-auto-retrans-threshold=<int>                   ; quantas retransmissões de requisição tornam a tentativa falha (padrão : 3)
 --hostlist-auto-debug=<logfile>                           ; depura detecções positivas da autohostlist
 --new                                                     ; inicia nova estratégia (novo perfil)
 --skip                                                    ; não usar este perfil
 --filter-l3=ipv4|ipv6                                     ; filtro de protocolo L3. múltiplos valores separados por vírgula.
 --filter-tcp=[~]port1[-port2]|*                           ; filtro de porta TCP. ~ significa negação. definir tcp sem definir udp nega udp. lista separada por vírgula suportada.
 --filter-udp=[~]port1[-port2]|*                           ; filtro de porta UDP. ~ significa negação. definir udp sem definir tcp nega tcp. lista separada por vírgula suportada.
 --filter-l7=<proto>                                       ; filtro de protocolo L6-L7. múltiplos valores separados por vírgula. proto: http tls quic wireguard dht discord stun unknown
 --filter-ssid=ssid1[,ssid2,ssid3,...]                     ; filtro de SSID wifi por perfil
 --ipset=<filename>                                        ; filtro de inclusão ipset (um ip/CIDR por linha, ipv4 e ipv6 aceitos, gzip suportado, múltiplos ipsets permitidos)
 --ipset-ip=<ip_list>                                      ; lista fixa de sub-redes separadas por vírgula
 --ipset-exclude=<filename>                                ; filtro de exclusão ipset (um ip/CIDR por linha, ipv4 e ipv6 aceitos, gzip suportado, múltiplos ipsets permitidos)
 --ipset-exclude-ip=<ip_list>                              ; lista fixa de sub-redes separadas por vírgula
```

Muitos parâmetros que lidam com dados binários aceitam carregamento a partir de uma string hex prefixada por "0x" ou de um arquivo.
O nome do arquivo pode ser usado como está ou prefixado com "@". Se houver um prefixo "+número" antes de "@", ele indica o deslocamento (offset) dos dados dentro do arquivo.
O deslocamento deve ser menor que o tamanho dos dados.

### Ataque de dessincronização do DPI

A ideia é pegar a mensagem original, modificá-la e adicionar informações falsas extras de forma que o sistema operacional do servidor aceite apenas os dados originais,
mas o DPI não consiga reconstruir a mensagem original ou veja algo que não consiga identificar como uma requisição proibida.

Existe um conjunto de ferramentas para atingir esse objetivo.
Podem ser pacotes falsos que chegam ao DPI mas não ao servidor, ou são rejeitados por ele, segmentação TCP ou fragmentação IP.
Existem ataques baseados em números de sequência TCP. Os métodos podem ser combinados de várias maneiras.

### Fakes

Fakes são pacotes gerados separadamente pelo nfqws que carregam informações falsas para o DPI. Eles não devem chegar ao servidor ou devem ser rejeitados por ele. Caso contrário, a conexão TCP ou o fluxo de dados seria quebrado. Há várias formas de resolver isso.

* **md5sig** não funciona em todos os servidores. Normalmente funciona apenas em servidores Linux. A opção TCP MD5 exige espaço extra no cabeçalho TCP
  e pode causar estouro de MTU durante fakedsplit/fakeddisorder em posições baixas quando uma requisição multissegmentada (TLS kyber) é transmitida.
  O `nfqws` não consegue redistribuir dados entre os segmentos TCP originais. O erro exibido é 'message too long'.
* **badsum** não funciona se seu dispositivo estiver atrás de um NAT que não deixa passar pacotes inválidos.
  A configuração comum de roteador Linux NAT não os deixa passar. A maioria dos roteadores domésticos é baseada em Linux.
  A configuração padrão do sysctl `net.netfilter.nf_conntrack_checksum=1` faz o conntrack verificar os checksums de tcp e udp
  e marcar como INVALID os pacotes com checksum inválido.
  Tipicamente, as regras iptables incluem uma regra para descartar pacotes com estado INVALID na cadeia FORWARD.
  A combinação desses fatores impede que pacotes badsum passem pelo roteador.
  No OpenWRT o sysctl mencionado vem configurado como 0 de fábrica; em outros roteadores costuma ficar no padrão "1".
  Para o nfqws funcionar corretamente através do roteador, defina `net.netfilter.nf_conntrack_checksum=0` no roteador.
  O sistema nunca verifica checksums de pacotes gerados localmente, então o nfqws sempre funcionará no próprio roteador.
  Se você estiver atrás de outro NAT, como o do ISP, e ele não deixa passar pacotes inválidos, não há nada a fazer.
  Mas normalmente os ISPs deixam passar badsum.
  Alguns adaptadores/switches/drivers habilitam filtragem por hardware de badsum na recepção, não deixando o pacote chegar ao sistema operacional.
  Isso foi observado em um dispositivo baseado em Mediatek MT7621.
  Tentou-se modificar o driver ethernet da MediaTek sem sucesso, provavelmente é uma limitação imposta pelo hardware.
  Ainda assim, o dispositivo permitia enviar pacotes badsum; o problema só existia para tráfego de passagem dos clientes.
* **badseq** os pacotes serão descartados pelo servidor, mas o DPI também pode ignorá-los.
  O incremento padrão de badseq é -10000 porque alguns DPIs descartam pacotes fora de uma pequena janela TCP.
  Mas isso também pode causar problemas quando `--dpi-desync-any-protocol` está habilitado.
  Para ter 100% de certeza de que o pacote falso não cabe na janela TCP do servidor, considere definir o incremento de badseq como 0x80000000
* **TTL** parece a melhor opção, mas exige ajuste específico para cada ISP. Se o DPI estiver mais distante que os sites locais do ISP
  você pode cortar o acesso a eles. É necessária uma lista manual de exclusão de IPs. É possível usar md5sig junto com ttl.
  Dessa forma você não prejudica nada, mas tem boas chances de ajudar a abrir os sites locais do ISP.
  Se não for possível encontrar uma solução automática, use `zapret-hosts-user-exclude.txt`.
  Alguns firmwares de roteador corrigem o TTL de saída. Sem desativar essa opção, o engano por TTL não funcionará.
* **hopbyhop** é apenas para ipv6. Esse engano adiciona um cabeçalho de extensão vazio `hop-by-hop options` ou dois cabeçalhos no caso de `hopbyhop2`.
  Pacotes com dois cabeçalhos hop-by-hop violam a RFC e são descartados por todos os sistemas operacionais.
  Todos os sistemas operacionais aceitam pacotes com um cabeçalho hop-by-hop.
  Alguns ISPs/operadores descartam pacotes ipv6 com opções hop-by-hop. Os fakes também não serão processados pelo servidor,
  seja porque o ISP os descarta ou porque há dois cabeçalhos iguais.
  DPIs ainda podem analisar pacotes com um ou dois cabeçalhos hop-by-hop.
* **datanoack** envia fakes TCP sem a flag ACK. Servidores não aceitam isso, mas o DPI pode aceitar.
  Esse modo pode quebrar o NAT e pode não funcionar com iptables se masquerade for usado, mesmo a partir do próprio roteador.
  Funciona corretamente com nftables. Provavelmente exige um endereço IP externo (alguns ISPs deixam esses pacotes passarem pelo NAT deles).
* Manipule as **flags tcp** com `--dpi-desync-tcp-flags-set` e `--dpi-desync-tcp-flags-unset`.
  Uma combinação inválida de flags tcp pode fazer o servidor descartar o pacote, mas o DPI pode aceitá-lo.
  Por exemplo, defina SYN nos fakes. Isso pode não funcionar em todos os servidores.
  `datanoack` pode ser substituído por `--dpi-desync-tcp-flags-unset=ACK`.
* **ts** adiciona ao TSval o valor de incremento ts (-600000 por padrão). Servidores descartam pacotes com TSval em certa faixa.
  Testes práticos sugerem um incremento entre -100 e -0x80000000.
  Timestamps são gerados pelo sistema operacional do cliente. No Linux eles estão habilitados por padrão. No Windows, por padrão, estão desabilitados.
  Podem ser habilitados com o comando : `netsh interface tcp set global timestamps=enabled` .
  O engano por ts exige que os timestamps estejam habilitados. Eles devem estar habilitados em todos os sistemas clientes.
  TSecr permanece sem modificação.
* **autottl** tenta adivinhar automaticamente a contagem de saltos até o servidor e calcular o TTL adicionando um valor delta, que pode ser positivo ou negativo.
  Deltas positivos devem ser precedidos por um sinal unário `+`. Deltas sem sinal unário são tratados como negativos por compatibilidade com versões antigas.
  Essa técnica depende dos valores padrão de TTL bem conhecidos usados pelos sistemas operacionais : 64,128,255.
  O nfqws precisa do primeiro pacote recebido para ver o TTL dele. Você deve redirecioná-lo também.
  Se o TTL resultante ficar fora da faixa (min,max), ele é normalizado para min ou max.
  Se o delta for negativo e o TTL for maior que a contagem de saltos estimada, ou o delta for positivo e o TTL for menor que a contagem estimada,
  então o autottl falha e volta ao valor fixo.
  Isso pode ajudar se houver múltiplos DPIs em canais de backbone, não apenas perto do ISP.
  Pode falhar se os caminhos de entrada e saída não forem simétricos.


`--dpi-desync-fooling` aceita múltiplos valores separados por vírgula.


Múltiplos parâmetros `--dpi-desync-fake-???` são suportados, exceto `--dpi-desync-fake-syndata`.
Os fakes são enviados na ordem especificada. `--dpi-desync-repeats` reenvia cada fake.
A ordem resultante será : `fake1 fake1 fake1 fake2 fake2 fake2 fake3 fake3 fake3 .....`


### Modificações de fakes

Por padrão, todos os fakes tcp são enviados com a mesma sequência do pacote original.
Isso pode ser alterado com `--dpi-desync-fake-tcp-mod=seq`. Nesse caso, todos os fakes são enviados como se fossem segmentos tcp de um único fake.

O **nfqws** tem um fake TLS embutido. Ele pode ser personalizado com a opção `--dpi-desync-fake-tls`.
Os dados falsos personalizados podem ser qualquer coisa — um TLS Client Hello válido ou dados arbitrários.
É possível usar um TLS Client Hello com qualquer fingerprint e qualquer SNI.

O **nfqws** pode realizar algumas modificações em fakes válidos de TLS Client Hello em tempo de execução com a opção `--dpi-desync-fake-tls-mod`.

  * `none`. Não faz nenhuma modificação.
  * `rnd`. Randomiza os campos `random` e `session id`. Aplicado em cada requisição.
  * `rndsni`. Randomiza o SNI. Se o SNI tiver >=7 caracteres, aplica um SLD aleatório com um TLD conhecido. Caso contrário, preenche com caracteres aleatórios. Aplicado apenas uma vez na inicialização.
  * `dupsid`. Copia o `session ID` do TLS Client Hello original. Tem precedência sobre `rnd`. Aplicado em cada requisição.
  * `sni=<sni>`. Define o valor de SNI especificado. Altera o tamanho do fake TLS e corrige os comprimentos na estrutura TLS. Aplicado uma vez na inicialização, antes de `rndsni`.
  * `padencap`. A extensão de padding é estendida pelo tamanho do TLS Client Hello original (incluindo variações com múltiplos pacotes e kyber). A extensão de padding é adicionada ao final se não existir; caso contrário, deve ser a última extensão. Todos os comprimentos são aumentados. O tamanho do fake não muda. Pode ser útil se o DPI não analisa corretamente os números de sequência. Aplicado em cada requisição.

Por padrão, se um fake personalizado não for definido, os mods `rnd,rndsni,dupsid` são aplicados. Se definido - `none`.
Esse comportamento é compatível com versões anteriores, com a adição de `dupsid`.

Se houver múltiplos fakes TLS, cada um recebe o último mod.
Se um mod for especificado depois de um fake, ele substitui o mod anterior.
Dessa forma é possível usar mods diferentes para cada fake TLS.

Se um mod for definido para um fake não-TLS, ocorre um erro. Use `--dpi-desync-fake-tls-mod=none`.

Exemplo : `--dpi-desync-fake-tls=iana_org.bin --dpi-desync-fake-tls-mod=rndsni --dpi-desync-fake-tls=0xaabbccdd --dpi-desync-fake-tls-mod=none`

### Segmentação TCP

  * `multisplit`. divide a requisição nas posições especificadas em `--dpi-desync-split-pos`
  * `multidisorder`. igual a `multisplit`, mas envia em ordem reversa
  * `fakedsplit`. divisão sequencial em uma posição com mistura de fakes
  * `hostfakesplit` (altorder=0). finge a parte do host da requisição : antes do host, host falso aleatório, host real (opcionalmente divide esta parte), repetição de host falso aleatório, depois do host
  * `hostfakesplit` (altorder=1). finge a parte do host da requisição : antes do host, host falso aleatório, depois do host, host real (opcionalmente divide esta parte)
  * `fakedsplit`. divisão reversa em uma posição com mistura de fakes

`--dpi-desync-fakedsplit-mod=altorder=N` especifica um número que influencia a presença de fakes individuais em `fakedsplit`/`fakeddisorder`.

Segmentos TCP de `fakedsplit` de mensagens com múltiplos pacotes com posição de divisão :

  * `altorder=0`. fake 1º segmento, 1º segmento, fake 1º segmento, fake 2º segmento, 2º segmento, fake 2º segmento
  * `altorder=1`. 1º segmento, fake 1º segmento, fake 2º segmento, 2º segmento, fake 2º segmento
  * `altorder=2`. 1º segmento, fake 2º segmento, 2º segmento, fake 2º segmento
  * `altorder=3`. 1º segmento, fake 2º segmento, 2º segmento

Segmentos TCP de `fakeddisorder` de mensagens com múltiplos pacotes com posição de divisão :

  * `altorder=0`. fake 2º segmento, 2º segmento, fake 2º segmento, fake 1º segmento, 1º segmento, fake 1º segmento
  * `altorder=1`. 2º segmento, fake 2º segmento, fake 1º segmento, 1º segmento, fake 1º segmento
  * `altorder=2`. 2º segmento, fake 1º segmento, 1º segmento, fake 1º segmento
  * `altorder=3`. 1º segmento, fake 1º segmento, 1º segmento

Segmentos TCP de `fakedsplit`/`fakeddisorder` de mensagens com múltiplos pacotes sem posição de divisão :

  * `altorder=0`. fake, original, fake
  * `altorder=8`. original, fake
  * `altorder=16`. original

O `altorder=N` resultante é a soma das duas partes `altorder` mencionadas acima.

`--dpi-desync-fakedsplit-pattern` define o payload dos fakes em `fakedsplit`/`fakeddisorder`. Por padrão, o padrão é simplesmente `0x00`.
O deslocamento da parte dividida + o deslocamento do pacote atual na mensagem de múltiplos pacotes definem o deslocamento no padrão.

As posições são definidas por marcadores.

* **Marcador absoluto positivo** — deslocamento numérico dentro de um pacote ou grupo de pacotes a partir do início
* **Marcador absoluto negativo** — deslocamento numérico dentro de um pacote ou grupo de pacotes a partir do byte seguinte ao fim
* **Marcador relativo** — deslocamento positivo ou negativo relativo a uma posição lógica dentro de um pacote ou grupo de pacotes

Posições relativas :

* **method** — início do método HTTP ('GET', 'POST', 'HEAD', ...). O método costuma estar sempre na posição 0, mas pode se deslocar por causa do engano `--methodeol`. Se enganado, a posição pode virar 1 ou 2.
* **host** — início do hostname em um protocolo conhecido (http, TLS)
* **endhost** — o byte seguinte ao último byte do hostname
* **sld** — início do domínio de segundo nível no hostname
* **endsld** — o byte seguinte ao último byte do SLD
* **midsld** — meio do SLD
* **sniext** — início do campo de dados na extensão SNI do TLS. Qualquer extensão tem 2 bytes de tipo e comprimento seguidos pelo campo de dados.

Exemplo de lista de marcadores : `100,midsld,sniext+1,endhost-2,-10`.

Ao dividir, todos os marcadores são resolvidos para deslocamentos absolutos. Se uma posição relativa não existir no protocolo atual, ela é descartada. Em seguida, todos os deslocamentos resolvidos são normalizados para o deslocamento do pacote atual no grupo de múltiplos pacotes (TLS com kyber, por exemplo). Posições fora do pacote atual são descartadas. As posições restantes são ordenadas e deduplicadas.

No caso de `multisplit` ou `multidisorder`, a divisão é cancelada se não restar nenhuma posição.

`fakedsplit` e `fakeddisorder` usam apenas uma posição de divisão. Ela é buscada na lista `--dpi-desync-split-pos` por um algoritmo especial.
Primeiro são buscados marcadores relativos. Se não houver nenhum adequado, são buscados marcadores absolutos. Se nada for encontrado, usa-se a posição 1.

Por exemplo, `--dpi-desync-split-pos=method+2,midsld,5` significa `method+2` para http, `midsld` para TLS e 5 para os demais.

`--dpi-desync-fakedsplit-mod=altorder=N` muda o `fakedsplit` para ordem alternada de segmentos.

`hostfakesplit` finge apenas a parte do hostname da requisição, dificultando distinguir entre nomes de host reais e falsos.
Funciona para protocolos TCP com host : TLS e HTTP. O hostname real pode ser dividido adicionalmente usando o marcador `--dpi-desync-hostfakesplit-midhost`.
Por exemplo, `--dpi-desync-hostfakesplit-midhost=midsld`. A posição deve estar dentro da faixa do host, ou a divisão não acontecerá.
Requisições com múltiplos pacotes são suportadas se a parte do hostname ainda não estiver dividida. Se estiver, o engano é cancelado.

Por padrão, nomes de host falsos são gerados aleatoriamente em tempo real usando o padrão `[0-9a-z]`. Se o comprimento do host for >= 7, um ponto é inserido para simular um TLD de 3 caracteres e os últimos 3 caracteres são substituídos por um TLD aleatório conhecido de 3 caracteres.
É possível definir um modelo de host falso : `--dpi-desync-hostfakesplit-mod=host=<hostname>`.
O hostname do modelo é expandido para a esquerda até o tamanho do hostname original com caracteres aleatórios do padrão `[0-9a-z]` : "www.networksolutions.com" -> "h8xmdba4tv7a8.google.com".
Se o hostname original for menor que o modelo, ele é cortado : "habr.com" -> "ogle.com".
Se o hostname original for um caractere maior que o modelo, um ponto é anexado à esquerda : "www.xxx.com" => ".google.com".
Por isso é uma boa ideia usar hostnames curtos no modelo : "ya.ru", "vk.com", "x.com".

`--dpi-desync-hostfakesplit-mod=altorder=1` muda o `hostfakesplit` para ordem alternada de segmentos. `altorder=1` envia a requisição inteira com o host falso sequencialmente e, depois, o segmento do host real.

### Sobreposição de números de sequência

`seqovl` adiciona bytes `seqovl` ao início de um dos segmentos originais e diminui o número de sequência. Para `split` — ao primeiro segmento; para `disorder` — ao início do penúltimo segmento enviado (o segundo na sequência original).

No modo `split` isso cria um pacote parcialmente dentro da janela. O sistema operacional recebe apenas a parte dentro da janela.
No modo `disorder` o sistema operacional recebe a parte falsa e a parte real do segundo segmento, mas não entrega os dados ao soquete até que o primeiro segmento seja recebido. O primeiro segmento sobrescreve a parte falsa do segundo segmento. Em seguida, o sistema operacional entrega os dados originais ao soquete.
Todos os sistemas operacionais Unix, exceto Solaris, preservam os últimos dados recebidos. Isso não vale para servidores Windows, e `disorder` com `seqovl` não funcionará.
Disorder exige que `seqovl` seja menor que a posição de divisão. Caso contrário, `seqovl` não é possível e será cancelado.
O método permite evitar fakes separados. Fakes e dados reais são misturados.

### Atribuição de IP_ID

Alguns DPIs verificam o valor de ip_id de ipv4. Normalmente o sistema operacional incrementa o ip_id a cada pacote. Alguns softwares anti-DPI podem enviar fakes ou segmentos TCP com o mesmo ip_id do original, causando o gatilho de bloqueio.

A sequência de ip_id será quebrada caso fakes ou segmentos TCP adicionais sejam enviados, porque o sistema operacional não sabe nada sobre eles. Ainda assim existem opções para atribuir ip_id aos pacotes gerados.

O parâmetro `ip-id` define o esquema de atribuição de ip_id para um perfil de dessincronização :

  * `seq` (padrão) : incrementa ip_id para cada pacote seguinte. No caso de `multidisorder`, aumenta o ip_id pelo número de segmentos TCP e depois diminui 1 por pacote.
  * `seqgroup` : igual a `seq`, mas envia substitutos falsos com o mesmo ip_id das partes originais. Relacionado apenas a segmentos TCP falsos com o mesmo tamanho e a mesma sequência dos originais.
  * `rnd` : atribui ip_id aleatório
  * `zero` : sempre define zero. Linux e BSD enviarão zero; o Windows substituirá zero pelo próprio contador.

O cabeçalho ipv6 não tem campo ip_id; o parâmetro `ip-id` é ignorado para ipv6.

### Modos específicos de ipv6

Os modos de dessincronização `hopbyhop`, `destopt` e `ipfrag1` (não são a mesma coisa que o engano `hopbyhop` !) são apenas para ipv6. Um cabeçalho `hop-by-hop`, `destination options` ou `fragment` é adicionado a todos os pacotes dessincronizados.
O cabeçalho extra aumenta o tamanho do pacote e não pode ser aplicado a pacotes de tamanho máximo.
Se não for possível enviar o pacote modificado, o original será enviado.
A ideia é que o DPI veja 0 no campo next header do cabeçalho ipv6 principal e não percorra a cadeia de cabeçalhos de extensão até encontrar o cabeçalho de transporte.
Os modos `hopbyhop`, `destopt`, `ipfrag1` podem ser usados com qualquer modo de segunda fase, exceto `ipfrag1+ipfrag2`.
Por exemplo, `hopbyhop,multisplit` significa dividir o pacote TCP original em várias partes e adicionar um cabeçalho hop-by-hop a cada uma.
Com `hopbyhop,ipfrag2` a sequência de cabeçalhos será : `ipv6,hop-by-hop,fragment,tcp/udp`.
O modo `ipfrag1` pode não funcionar sempre sem preparações especiais. Veja as observações em "Fragmentação de IP".

### Modificação do original

Os parâmetros `--orig-ttl` e `--orig-ttl6` permitem definir o TTL dos pacotes originais.
Todas as manipulações posteriores de pacotes, como segmentação, usam o original modificado como fonte de dados e herdam o TTL modificado.

`--orig-autottl` e `--orig-autottl6` funcionam da mesma forma que `dpi-desync-autottl`, mas nos pacotes originais.
O delta deve ter o sinal unário `+` para produzir um TTL maior que a contagem de saltos estimada. Caso contrário, nada chegará ao servidor.
Exemplo : `--orig-autottl=+5:3-64`.

`--orig-mod-start` e `--orig-mod-cutoff` especificam as condições de início e fim da modificação do original. Funcionam da mesma forma que
`--dpi-desync-start` e `--dpi-desync-cutoff`.

Essa função pode ser útil quando o DPI caça fakes e bloqueia conexões suspeitas.
O DPI pode calcular a diferença de TTL entre pacotes e disparar o bloqueio se ela ultrapassar certo limite.

### Duplicatas

Duplicatas são cópias dos pacotes originais enviadas antes deles. São habilitadas por `--dup=N`, onde N é a quantidade de duplicatas.
`--dup-replace` desabilita o envio do original.

Duplicatas são enviadas apenas quando o original também seria enviado sem reconstrução.
Por exemplo, se ocorre segmentação TCP, o original é na verdade descartado e substituído por pacotes novos construídos artificialmente.
Nesse caso, duplicatas não são enviadas.

Todos os modos de engano de dup estão disponíveis : `--dup-ttl`, `--dup-ttl6`, `--dup-fooling`.
Você decide se esses pacotes precisam chegar ao servidor e em que forma, de acordo com a estratégia pretendida.

`--dup-autottl` e `--dup-autottl6` funcionam da mesma forma que `dpi-desync-autottl`.
O delta pode ser precedido por um sinal unário `+` ou `-`.
Exemplo : `--dup-autottl=-2:3-64`.

`--dup-start` e `--dup-cutoff` especificam as condições de início e fim da duplicação. Funcionam da mesma forma que
`--dpi-desync-start` e `--dpi-desync-cutoff`.

Essa função pode ajudar se o DPI compara certas características dos pacotes falsos e originais e bloqueia a conexão se elas diferirem de alguma forma.
Duplicatas enganadas podem convencer o DPI de que toda a sessão tem uma anomalia.
Por exemplo, toda a conexão é protegida por assinatura MD5, e não pacotes individuais.

### Reação à resposta do servidor

Existem DPIs que analisam as respostas do servidor, em particular o certificado do ServerHello que contém o(s) nome(s) de domínio. A confirmação da entrega do ClientHello é um pacote ACK do servidor com número de ACK correspondente ao tamanho do ClientHello+1.
Na variante disorder, normalmente chega primeiro um reconhecimento seletivo (SACK) e depois um ACK completo.
Se, em vez de ACK ou SACK, houver um pacote RST com atraso mínimo, o DPI te corta já na etapa da requisição.
Se o RST vier depois de um ACK completo com atraso de cerca de um ping até o servidor, provavelmente o DPI age com base na resposta do servidor. O DPI pode ficar satisfeito com um bom ClientHello e parar de monitorar a sessão TCP sem verificar o ServerHello. Nesse caso você teve sorte. A opção 'fake' pode funcionar.
Se ele não parar de monitorar e continuar verificando o ServerHello, o parâmetro `--wssize` pode ajudar (veja [CONNTRACK](#conntrack)).
Caso contrário, é difícil superar isso sem a ajuda do servidor.
A melhor solução é habilitar o suporte a TLS 1.3 no servidor. O TLS 1.3 envia o certificado do servidor de forma criptografada.
Essa é uma recomendação a todos os administradores de sites bloqueados. Habilitem o TLS 1.3. Vocês darão mais oportunidades para burlar o DPI.

### Modo SYNDATA

Normalmente os SYNs vêm sem payload de dados. Se houver payload, ele é ignorado pelos principais sistemas operacionais quando TCP fast open (TFO) não está envolvido, mas pode não ser ignorado pelo DPI.
Conexões originais com TFO não são tocadas, porque isso as quebraria com certeza.
Sem parâmetros extras, o payload é de 16 bytes zero.

### Combinações de dessincronização do DPI

`--dpi-desync` aceita até 3 modos separados por vírgula.

* Modos de fase 0 atuam durante o estabelecimento da conexão : `synack`, `syndata` `--wsize`, `--wssize`. Os filtros de [hostlist](#múltiplas-estratégias) são aplicáveis apenas se [`--ipcache-hostname`](#cache-de-ip) estiver habilitado.
* Na 1ª fase, fakes são enviados antes dos dados originais : `fake`, `rst`, `rstack`.
* Na 2ª fase, os dados originais são enviados de forma modificada (por exemplo `fakedsplit` ou `ipfrag2`).

Os modos devem ser especificados em ordem crescente de fase.

### Cache de IP

`ipcache` é a estrutura na memória do processo que armazena algumas informações por chave de endereço IP e nome de interface.
Essas informações podem ser usadas como dados ausentes. Atualmente são usadas nos seguintes casos :

1. IP,interface => contagem de saltos. Isso é usado para aplicar autottl na fase 0 desde o primeiro pacote da sessão. Se o registro não existir, o autottl não será aplicado imediatamente. Na segunda vez será aplicado imediatamente usando a contagem de saltos em cache.

2. IP => hostname. O hostname é armazenado em cache para uso em estratégias de fase 0. O modo é desabilitado por padrão e pode ser habilitado pelo parâmetro `ipcache-hostname`.
Essa técnica é experimental. Não há correspondência um a um entre IP e nome de domínio. Múltiplos domínios podem resolver para o mesmo IP.
Se ocorrer colisão, o hostname é substituído. Em CDNs, um domínio pode resolver para IPs diferentes ao longo do tempo. `--ipcache-lifetime` limita por quanto tempo o registro em cache é válido. O padrão é 2 horas.
Esteja preparado para resultados inesperados que só podem ser explicados lendo os logs de depuração.

SIGUSR2 força o processo a exibir seu ipcache na saída padrão.

### CONNTRACK

O nfqws é equipado com um sistema minimalista de rastreamento de conexões (conntrack).
Ele é usado quando certos métodos específicos de burla do DPI são empregados e ajuda a remontar requisições de múltiplos pacotes.

O conntrack pode rastrear a fase da conexão : SYN,ESTABLISHED,FIN, contagens de pacotes nas duas direções e números de sequência.

Ele pode receber pacotes unidirecionais ou bidirecionais.

Um pacote SYN ou SYN,ACK cria uma entrada na tabela do conntrack.

Por isso o redirecionamento do iptables deve começar com o primeiro pacote, embora possa ser cortado depois usando o filtro connbytes.

O primeiro pacote UDP visto cria um fluxo UDP. Ele define a direção do fluxo. Em seguida, todos os pacotes com o mesmo
`src_ip,src_port,dst_ip,dst_port` são considerados parte do mesmo fluxo UDP. O fluxo UDP existe até o timeout de inatividade.

Uma conexão é removida da tabela assim que não for mais necessária para as necessidades do nfqws ou quando ocorre um timeout.

Existem 3 timeouts para cada estado de conexão. Podem ser alterados no parâmetro `--ctrack-timeouts`.

`--wssize` altera o tamanho da janela TCP do servidor para forçá-lo a enviar respostas divididas.
Para que isso afete todos os sistemas operacionais de servidor, é necessário alterar o tamanho da janela em cada pacote de saída
antes de enviar a mensagem cuja resposta precisa ser dividida (por exemplo, TLS ClientHello).
Por isso o conntrack é necessário para saber quando parar de aplicar o tamanho pequeno de janela.

Se você não parar e manter o wssize baixo o tempo todo, a velocidade cairá catastroficamente.
O Linux consegue contornar isso com o filtro connbytes, mas outros sistemas operacionais podem não ter um filtro similar.

No caso http(s), o wssize para depois da primeira requisição http ou do TLS ClientHello, a menos que `--wssize-forced-cutoff=0` seja especificado.

Se você lida com um protocolo não-http(s), precisa de `--wssize-cutoff`. Ele define o limite onde o wssize para.

O limite pode ser prefixado com 'n' (número de pacote começando em 1), 'd' (número de pacote de dados começando em 1),
's' (número de sequência relativa — bytes enviados pelo cliente + 1).

Se um pacote de requisição http ou TLS ClientHello for detectado, o wssize para imediatamente, ignorando a opção wssize-cutoff.
Essa ação é chamada de "forced wssize cutoff" e pode ser desabilitada com `--wssize-forced-cutoff=0`.

Se o seu protocolo tende a longos períodos de inatividade, aumente o timeout da fase ESTABLISHED com `--ctrack-timeouts`.

O timeout padrão é baixo — apenas 5 minutos.

Não esqueça que o nfqws é alimentado com pacotes redirecionados. Se você limitar o redirecionamento com connbytes,
entradas ESTABLISHED podem permanecer na tabela até serem removidas por timeout.

Para diagnosticar o estado do conntrack, envie o sinal SIGUSR1 ao nfqws : `killall -SIGUSR1 nfqws`.

O nfqws exibirá a tabela atual do conntrack na saída padrão.

Tipicamente, em um pacote SYN o cliente envia, além do tamanho da janela, a extensão TCP **fator de escala** (scaling factor).
O fator de escala é a potência de dois pela qual o tamanho da janela é multiplicado : 0=>1, 1=>2, 2=>4, ..., 8=>256, ...

O parâmetro wssize especifica o fator de escala depois dos dois pontos.

O fator de escala só pode diminuir; o aumento é bloqueado para impedir que o servidor exceda o tamanho da janela do cliente.

Para forçar um servidor TLS a fragmentar a mensagem ServerHello e evitar a detecção do hostname pelo DPI, use `--wssize=1:6`

A regra principal é definir scale_factor o maior possível, de modo que, após a recuperação, o tamanho final da janela seja o máximo possível. Se você definir `scale_factor` como 64:0, será muito lento.

Por outro lado, a resposta do servidor não deve ser grande o suficiente para que o DPI encontre o que procura.

`--wssize` não é aplicado em perfis de dessincronização com filtro hostlist, porque atua desde o início da conexão, quando ainda não é possível extrair o nome do host. Mas funciona com perfis de autohostlist.

`--wssize` pode deixar sites mais lentos e/ou aumentar o tempo de resposta. É desejável usar outros métodos, se possível.

`--dpi-desync-cutoff` permite definir o limite em que a aplicação do dpi-desync para.
Pode ser prefixado com 'n', 'd' ou 's', da mesma forma que `--wssize-cutoff`.
Útil com `--dpi-desync-any-protocol=1`.
Se a conexão sair do conntrack e `--dpi-desync-cutoff` estiver definido, o `dpi desync` não será aplicado.

Defina os timeouts do conntrack adequadamente.

### Remontagem (Reassemble)

O nfqws suporta a remontagem de ClientHello de TLS e QUIC.
Eles podem consistir em múltiplos pacotes se a criptografia kyber for usada (padrão a partir do chromium 124).
O Chromium randomiza o fingerprint TLS. O SNI pode estar em qualquer pacote ou entre eles.
DPIs com estado geralmente remontam todos os pacotes da requisição e então aplicam a decisão de bloqueio.
Se o nfqws recebe um ClientHello parcial, ele inicia uma sessão de remontagem. Os pacotes são retidos até que ela termine.
Em seguida, passam pela dessincronização usando a mensagem completamente remontada.
Em qualquer erro, a remontagem é cancelada e todos os pacotes retidos são enviados imediatamente sem dessincronização.

Há suporte especial para todas as opções de divisão TCP para TLS com múltiplos segmentos. A posição de divisão é tratada como orientada à mensagem, não ao pacote. Por exemplo, se seu cliente envia um TLS ClientHello de tamanho 2000 e o SNI está em 1700, com modo de dessincronização `fake,multisplit`, o fake é enviado primeiro, depois o primeiro segmento original e o último segmento dividido. 3 segmentos no total.

### Suporte a UDP

Os ataques UDP são limitados. Não é possível fragmentar UDP no nível de transporte, apenas no nível de rede (ip).
Apenas os modos de dessincronização `fake`,`fakeknown`,`hopbyhop`,`destopt`,`ipfrag1`,`ipfrag2`,`udplen` e `tamper` são aplicáveis.
`fake`,`fakeknown`,`hopbyhop`,`destopt`,`ipfrag1` são modos da 1ª fase; os demais são da 2ª fase.
Como sempre, é possível combinar um modo da 1ª fase com um da 2ª fase, mas não é possível misturar modos da mesma fase.

`udplen` aumenta o tamanho do payload udp em `--dpi-desync-udplen-increment` bytes. O preenchimento é feito com zeros por padrão, mas pode ser substituído por um padrão.
Essa opção pode resistir a DPIs que monitoram os tamanhos dos pacotes UDP de saída.
Exige que o protocolo da aplicação não dependa do tamanho do payload udp.

Pacotes QUIC initial são reconhecidos. A descriptografia e a extração do hostname são suportadas, então o parâmetro `--hostlist` funcionará.
Handshakes de inicialização do Wireguard, DHT, STUN e pacotes de [IP Discovery de voz do Discord](https://discord.com/developers/docs/topics/voice-connections#ip-discovery) também são reconhecidos.
Para dessincronizar outros protocolos, use `--dpi-desync-any-protocol`.

O conntrack suporta udp. `--dpi-desync-cutoff` funcionará. O timeout do conntrack UDP pode ser definido no 4º parâmetro de `--ctrack-timeouts`.

O ataque de fake só é útil para DPIs com estado e é inútil para DPIs sem estado que tratam cada pacote isoladamente.
Por padrão, o payload falso é de 64 zeros. Pode ser substituído com `--dpi-desync-fake-unknown-udp`.

### Fragmentação de IP

Redes modernas podem ser muito hostis à fragmentação de IP. Pacotes fragmentados frequentemente não são entregues ou são refragmentados/remontados no caminho.
A posição de fragmentação é definida independentemente para tcp e udp. Por padrão, 24 e 8; deve ser múltiplo de 8.
O deslocamento começa no cabeçalho de transporte.

Fragmentos tcp quase sempre são filtrados. Não são adequados para sites arbitrários de forma alguma.
Fragmentos udp têm boas chances de sobreviver, mas não em todo lugar. É bom assumir uma taxa de sucesso no QUIC entre 50..75%.
Provavelmente maior com seu VPS. Às vezes são filtrados por proteção DDoS.

Existem nuances importantes ao trabalhar com fragmentos no Linux.

ipv4 : o Linux permite enviar fragmentos ipv4, mas as regras de firewall padrão na cadeia OUTPUT podem fazer o envio raw falhar.

ipv6 : não há como uma aplicação enviar fragmentos de forma confiável sem desfragmentação pelo conntrack.
Às vezes funciona, às vezes o sistema desfragmenta os pacotes.
Parece que kernels < 4.16 não têm forma simples de resolver esse problema. Descarregar o módulo `nf_conntrack`
e sua dependência `nf_defrag_ipv6` ajuda, mas isso impacta severamente a funcionalidade.
Kernels 4.16+ excluem da desfragmentação pacotes não rastreados.
Veja o código de `blockcheck.sh` como exemplo.

Às vezes é necessário carregar o módulo do kernel `ip6table_raw` com o parâmetro `raw_before_defrag=1`.
No OpenWRT, parâmetros de módulo são especificados depois dos nomes, separados por espaço, em arquivos em `/etc/modules.d`.

No Linux tradicional, verifique se `iptables-legacy` ou `iptables-nft` está em uso. Se for legacy, crie o arquivo
`/etc/modprobe.d/ip6table_raw.conf` com o seguinte conteúdo :
```
options ip6table_raw raw_before_defrag=1
```
Em algumas distribuições Linux é possível mudar o ip6tables atual com este comando: `update-alternatives --config ip6tables`.
Se você quiser permanecer com `nftables-nft`, precisa aplicar patch e recompilar sua versão.
Em `nft.c` encontre :
```
			{
				.name	= "PREROUTING",
				.type	= "filter",
				.prio	= -300,	/* NF_IP_PRI_RAW */
				.hook	= NF_INET_PRE_ROUTING,
			},
			{
				.name	= "OUTPUT",
				.type	= "filter",
				.prio	= -300,	/* NF_IP_PRI_RAW */
				.hook	= NF_INET_LOCAL_OUT,
			},
```
e substitua -300 por -450.

Isso deve ser feito manualmente; `blockcheck.sh` não consegue corrigir isso automaticamente para você.

Ou simplesmente migre para `nftables`. Lá você pode criar hooks com qualquer prioridade.

Parece não haver como fazer ipfrag com iptables para tráfego encaminhado quando há NAT.
`MASQUERADE` é um alvo terminal; depois dele o `NFQUEUE` não funciona.
O nfqws vê pacotes com endereço de origem da rede interna. Se fragmentado, o NAT não os processa.
Isso resulta em uma tentativa de enviar pacotes à internet com IP interno.
Você precisa usar nftables com prioridade de hook 101 ou superior.

### Múltiplas estratégias

O **nfqws** pode aplicar estratégias diferentes a requisições diferentes. Isso é feito com múltiplos perfis de dessincronização.
Os perfis são delimitados pelo parâmetro `--new`. O primeiro perfil é criado automaticamente e não exige `--new`.
Cada perfil tem um filtro. Por padrão ele está vazio e o perfil corresponde a qualquer pacote.
O filtro pode ter parâmetros rígidos : versão de ip, ipset e faixa de porta tcp/udp.
Parâmetros rígidos são sempre identificados sem ambiguidade, mesmo na fase zero, quando hostname e L7 ainda são desconhecidos.
Listas de hosts também podem atuar como filtro. Podem ser combinadas com parâmetros rígidos.
Quando um pacote chega, os perfis são verificados do primeiro ao último até que o primeiro filtro corresponda.
O filtro rígido é verificado primeiro. Se não corresponder, a verificação passa ao próximo perfil.
Se um perfil corresponde ao filtro rígido, ao filtro L7 e tem autohostlist, ele é selecionado imediatamente.
Se um perfil corresponde ao filtro rígido, ao filtro L7 e tem hostlist(s) normal(is), mas o hostname ainda é desconhecido, a verificação passa ao próximo perfil.
Caso contrário, as hostlists do perfil são verificadas quanto ao hostname. Se houver correspondência, o perfil é selecionado.
Caso contrário, a verificação passa ao próximo perfil.

É possível que, antes de conhecer L7 e o hostname, a conexão seja atendida por um perfil e, depois que essas informações são reveladas, mude para outro perfil.
Se você usa métodos de dessincronização de fase 0, pense com cuidado no que pode acontecer durante a troca de estratégia.
Use a logagem de `--debug` para entender melhor o que o **nfqws** faz.

Os perfis são numerados de 1 a N. Existe um último perfil vazio na cadeia, numerado 0.
Ele é usado quando nenhum filtro corresponde.

IMPORTANTE : múltiplas estratégias existem apenas para os casos em que não é possível combinar tudo em uma única estratégia.
Copiar e colar resultados de blockcheck de diferentes sites em múltiplas estratégias gera confusão.
Dessa forma você pode nunca desbloquear todos os recursos e apenas confundir a si mesmo.

IMPORTANTE : a implementação de ipset em modo usuário não foi projetada como substituto da versão do kernel. A versão do kernel é muito mais eficiente.
Ela é para sistemas que não têm suporte a ipset : Windows e Linux sem módulos de kernel nftables e ipset (Android, por exemplo).

### Filtragem por WIFI

O nome da interface wifi não está relacionado ao SSID conectado.
É possível conectar a interface a diferentes SSIDs.
Eles podem exigir estratégias diferentes. Como resolver isso?

Você pode iniciar e parar instâncias do nfqws manualmente. Mas também é possível automatizar.
A versão Windows, `winws`, tem o filtro global `--ssid-filter`.
Ele conecta ou desconecta o `winws` dependendo dos SSIDs conectados.
O roteamento não é levado em conta. Essa abordagem é possível porque o windivert pode ter múltiplos handlers com filtros sobrepostos.
Se o SSID muda, um `winws` conecta e os outros desconectam.

A solução do `winws` é difícil de implementar no Linux porque uma nfqueue pode ter apenas um handler e é impossível passar o mesmo tráfego para múltiplas filas.
Um deve conectar quando os outros já tenham desconectado.
Em vez disso, o `nfqws` tem o parâmetro por perfil `--filter-ssid`. Como `--ssid-filter`, ele aceita uma lista de SSIDs separada por vírgula.
O `nfqws` mantém uma lista ifname->SSID atualizada no máximo uma vez por segundo.
Quando um pacote chega, o nome da interface de entrada ou saída é comparado ao SSID e então usado no algoritmo de seleção de perfil.

A informação de SSID é obtida da mesma forma que `iw dev <ifname> info` (nl80211).
Infelizmente isso está quebrado desde o kernel 5.19 e ainda não foi corrigido no 6.14.
Nesse caso, usa-se o método `iwgetid` (wireless extensions).
Wireless extensions estão obsoletas. Alguns kernels podem ser compilados sem suporte a wext.
Antes de usar `--filter-ssid`, verifique se algum dos comandos mencionados consegue retornar o SSID.

### Máquinas virtuais

A maior parte da mágica de pacotes do nfqws não funciona em VMs do VirtualBox e VMware quando a rede está em NAT.
O hipervisor altera o TTL à força e não encaminha pacotes falsos.
Configure rede em bridge.

### IPTABLES para nfqws

> [!CAUTION]
> A partir do kernel Linux 6.17 existe o parâmetro CONFIG_NETFILTER_XTABLES_LEGACY, que não é habilitado por padrão. Muitas distribuições provavelmente não o habilitarão, tornando o iptables-legacy inoperante. Isso faz parte da descontinuação do iptables. Porém, o iptables-nft continua funcionando porque seu backend é baseado em nftables.

Esta é a forma comum de redirecionar parte do tráfego ao nfqws :

```
iptables -t mangle -I POSTROUTING -o <wan_interface> -p tcp -m multiport --dports 80,443 -m connbytes --connbytes-dir=original --connbytes-mode=packets --connbytes 1:6 -m mark ! --mark 0x40000000/0x40000000 -j NFQUEUE --queue-num 200 --queue-bypass
```

Essa variante funciona se o DPI tiver estado e não rastrear cada pacote separadamente em busca de "requisições ruins". Se ele não tiver estado, você precisa redirecionar todo o tráfego http plano de saída.

```
iptables -t mangle -I POSTROUTING -o <wan_interface> -p tcp --dport 443 -m connbytes --connbytes-dir=original --connbytes-mode=packets --connbytes 1:6 -m mark ! --mark 0x40000000/0x40000000 -j NFQUEUE --queue-num 200 --queue-bypass
iptables -t mangle -I POSTROUTING -o <wan_interface> -p tcp --dport 80 -m mark ! --mark 0x40000000/0x40000000 -j NFQUEUE --queue-num 200 --queue-bypass
```

O bit de mark é usado para evitar loops. O **nfqws** define essa mark em cada pacote injetado.
Também é necessário para a ordem correta dos pacotes injetados e para prevenir deadlocks.

`autottl` exige o pacote `SYN,ACK` de entrada ou o primeiro pacote de resposta (geralmente é o mesmo).

`autohostlist` precisa do `RST` e do `http redirect` de entrada.

É possível construir um filtro baseado em flags tcp e u32, mas connbytes é mais fácil.

`
iptables -t mangle -I PREROUTING -i <wan_interface> -p tcp -m multiport --sports 80,443 -m connbytes --connbytes-dir=reply --connbytes-mode=packets --connbytes 1:3 -m mark ! --mark 0x40000000/0x40000000 -j NFQUEUE --queue-num 200 --queue-bypass
`

Para QUIC :

```
iptables -t mangle -I POSTROUTING -o <wan_interface> -p udp --dport 443 -m connbytes --connbytes-dir=original --connbytes-mode=packets --connbytes 1:6 -m mark ! --mark 0x40000000/0x40000000 -j NFQUEUE --queue-num 200 --queue-bypass
```

6 pacotes cobrem possíveis retransmissões de QUIC initials e alimentam o modo `autohostlist`.

### NFTABLES para nfqws

Esta é a configuração inicial :

```
IFACE_WAN=wan

nft create table inet ztest

nft add chain inet ztest post "{type filter hook postrouting priority mangle;}"
nft add rule inet ztest post oifname $IFACE_WAN meta mark and 0x40000000 == 0 tcp dport "{80,443}" ct original packets 1-6 queue num 200 bypass
nft add rule inet ztest post oifname $IFACE_WAN meta mark and 0x40000000 == 0 udp dport 443 ct original packets 1-6 queue num 200 bypass

# auto hostlist evitando números ACK errados em pacotes RST,ACK enviados pelo DPI russo
sysctl net.netfilter.nf_conntrack_tcp_be_liberal=1 
nft add chain inet ztest pre "{type filter hook prerouting priority filter;}"
nft add rule inet ztest pre iifname $IFACE_WAN tcp sport "{80,443}" ct reply packets 1-3 queue num 200 bypass
```

Para usar `datanoack` ou `ipfrag` em tráfego de passagem, é necessária uma configuração POSTNAT especial. Os pacotes gerados devem ser marcados como **notrack** em estágio inicial para não serem invalidados pelo conntrack do Linux.

```
IFACE_WAN=wan

nft create table inet ztest

nft add chain inet ztest postnat "{type filter hook postrouting priority srcnat+1;}"
nft add rule inet ztest postnat oifname $IFACE_WAN meta mark and 0x40000000 == 0 tcp dport "{80,443}" ct original packets 1-6 queue num 200 bypass
nft add rule inet ztest postnat oifname $IFACE_WAN meta mark and 0x40000000 == 0 udp dport 443 ct original packets 1-6 queue num 200 bypass

nft add chain inet ztest predefrag "{type filter hook output priority -401;}"
nft add rule inet ztest predefrag "mark & 0x40000000 != 0x00000000 notrack"
```

Excluir a nftable :

```
nft delete table inet ztest
```

### Flow offloading (descarregamento de fluxo)

Se seu dispositivo suporta flow offloading (aceleração por hardware), iptables e nftables podem não funcionar. Com o offloading habilitado, os pacotes ignoram o fluxo padrão do netfilter. Ele deve ser desabilitado ou controlado seletivamente.

Kernels Linux mais novos têm software flow offloading (SFO). A situação é a mesma com SFO.

No `iptables`, o flow offloading é controlado pela extensão proprietária do OpenWRT `FLOWOFFLOAD`. `nftables` mais novos implementam suporte integrado a offloading.

O flow offloading não interfere com **tpws** e com tráfego `OUTPUT`. Ele só quebra o nfqws que engana tráfego `FORWARD`.

### Engano no lado do servidor

Também é possível.
O nfqws foi projetado para ataques no lado do cliente. Por isso ele reconhece tráfego direto e de resposta com base no papel no estabelecimento da conexão.
Se vê um SYN, o IP de origem é o IP do cliente. Se vê SYN,ACK, o IP de origem é o IP do servidor.
Para UDP, o endereço do cliente é considerado o IP de origem do primeiro pacote visto da tupla src_ip,src_port,dst_ip,dst_port.

Isso não funciona corretamente no lado do servidor. O tráfego do cliente é tráfego de resposta; o tráfego do servidor é tráfego direto.

`--wsize` funciona de qualquer forma. Pode ser usado tanto no cliente quanto no servidor.
Outras técnicas funcionam apenas se o nfqws tratar o tráfego como tráfego direto.
Para aplicá-las ao tráfego originado no servidor, desabilite o conntrack com o parâmetro `--ctrack-disable`.
Se um pacote não for encontrado no conntrack, ele é tratado como direto e técnicas como `multidisorder` serão aplicadas.

A maioria dos protocolos não será reconhecida porque o sistema de reconhecimento de protocolo só reage a pacotes do cliente.
Para fazer funcionar, use `--dpi-desync-any-protocol` com connbytes ou limitador de payload.
start/cutoff não estão disponíveis porque são baseados em conntrack.

`--synack-split` remove o pacote SYN,ACK padrão e o substitui por um pacote SYN, por SYN e ACK separados ou por ACK e SYN separados.
O cliente envia SYN,ACK em resposta, o que normalmente só o servidor faz.
Isso faz com que alguns DPIs interpretem errado os papéis no estabelecimento da conexão. Eles param de bloquear.
Veja [split handshake](https://nmap.org/misc/split-handshake.pdf).

No lado do servidor, o tráfego deve ser redirecionado ao nfqws usando números de porta de origem e a direção original de connbytes.


## tpws

tpws é um proxy transparente.

```
 @<config_file>                          ; lê arquivo de opções. deve ser o único argumento. outras opções são ignoradas.

 --debug=0|1|2|syslog|@<filename>        ; 1 e 2 significam logar no console e definir nível de depuração. para outros destinos use --debug-level.
 --debug-level=0|1|2                     ; especifica nível de depuração para syslog e @<filename>
 --dry-run                               ; valida os parâmetros e sai com código 0 se bem-sucedido
 --version                               ; imprime a versão e sai
 --bind-addr=<v4_addr>|<v6_addr>         ; para v6 link locals anexe %interface_name : fe80::1%br-lan
 --bind-iface4=<interface_name>          ; vincula ao primeiro endereço ipv4 da interface
 --bind-iface6=<interface_name>          ; vincula ao primeiro endereço ipv6 da interface
 --bind-linklocal=no|unwanted|prefer|force
                                          ; no : vincula apenas a ipv6 global
                                          ; unwanted (padrão) : prefere endereço global, depois LL
                                          ; prefer : prefere LL, depois global
                                          ; force : apenas LL
 --bind-wait-ifup=<sec>                  ; aguarda a interface aparecer e subir
 --bind-wait-ip=<sec>                    ; após ifup, aguarda até N segundos pelo endereço ip
 --bind-wait-ip-linklocal=<sec>          ; aceita apenas link locals nos primeiros N segundos, depois qualquer um
 --bind-wait-only                        ; aguarda as condições de vínculo e sai. código 0 se bem-sucedido.
 --connect-bind-addr=<v4_addr>|<v6_addr> ; endereço para conexões de saída. para v6 link locals anexe %%interface_name
 --port=<port>                           ; número da porta para escutar
 --socks                                 ; implementa proxy socks4/5 em vez de proxy transparente
 --local-rcvbuf=<bytes>                  ; SO_RCVBUF para trechos locais
 --local-sndbuf=<bytes>                  ; SO_SNDBUF para trechos locais
 --remote-rcvbuf=<bytes>                 ; SO_RCVBUF para trechos remotos
 --remote-sndbuf=<bytes>                 ; SO_SNDBUF para trechos remotos
 --nosplice                              ; não usa splice para transferir dados entre soquetes
 --skip-nodelay                          ; não define TCP_NODELAY para conexões de saída. incompatível com split.
 --local-tcp-user-timeout=<seconds>      ; define tcp user timeout para o trecho local (padrão : 10, 0 = padrão do sistema)
 --remote-tcp-user-timeout=<seconds>     ; define tcp user timeout para o trecho remoto (padrão : 20, 0 = padrão do sistema)
 --fix-seg=<int>                         ; recupera falhas de segmentação TCP ao custo de lentidão. aguarda até N mseg.
 --ipcache-lifetime=<int>                ; tempo em segundos para manter o nome de domínio em cache (padrão 7200). 0 = sem expiração
 --ipcache-hostname=[0|1]                ; 1 ou sem argumento habilita cache ip->hostname
 --no-resolve                            ; desabilita dns remoto do socks5
 --resolver-threads=<int>                ; número de threads de resolução
 --maxconn=<max_connections>             ; número máximo de trechos locais
 --maxfiles=<max_open_files>             ; máximo de descritores de arquivo (setrlimit). mínimo é (X*connections+16), onde X=6 no modo proxy tcp, X=4 no modo de modificação.
                                          ; vale reservar com multiplicador 1.5. por padrão maxfiles é (X*connections)*1.5+16
 --max-orphan-time=<sec>                 ; se o trecho local envia algo e fecha e o trecho remoto ainda está conectando, cancela a tentativa após N segundos

 --new                                   ; inicia nova estratégia (novo perfil)
 --skip                                  ; não usar este perfil
 --filter-l3=ipv4|ipv6                   ; filtro de protocolo L3. múltiplos valores separados por vírgula.
 --filter-tcp=[~]port1[-port2]|*         ; filtro de porta TCP. ~ significa negação. lista separada por vírgula suportada.
 --filter-l7=[http|tls|unknown]          ; filtro de protocolo L6-L7. múltiplos valores separados por vírgula.
 --ipset=<filename>                      ; filtro de inclusão ipset (um ip/CIDR por linha, ipv4 e ipv6 aceitos, gzip suportado, múltiplos ipsets permitidos)
 --ipset-ip=<ip_list>                    ; lista fixa de sub-redes separadas por vírgula
 --ipset-exclude=<filename>              ; filtro de exclusão ipset (um ip/CIDR por linha, ipv4 e ipv6 aceitos, gzip suportado, múltiplos ipsets permitidos)
 --ipset-exclude-ip=<ip_list>            ; lista fixa de sub-redes separadas por vírgula

 --hostlist=<filename>                   ; atua apenas em hosts da lista (um host por linha, subdomínios aplicados automaticamente se não prefixados com '^', gzip suportado, múltiplas listas permitidas)
 --hostlist-domains=<domain_list>        ; lista fixa de domínios separados por vírgula
 --hostlist-exclude=<filename>           ; não atuar em hosts da lista (um host por linha, subdomínios aplicados automaticamente se não prefixados com '^', gzip suportado, múltiplas listas permitidas)
 --hostlist-exclude-domains=<domain_list> ; lista fixa de domínios separados por vírgula
 --hostlist-auto=<filename>              ; detecta bloqueios de DPI e constrói a lista de hosts automaticamente
 --hostlist-auto-fail-threshold=<int>    ; quantas tentativas falhas adicionam o hostname à lista automática (padrão : 3)
 --hostlist-auto-fail-time=<int>         ; todas as falhas devem ocorrer nesse intervalo de segundos (padrão : 60)
 --hostlist-auto-debug=<logfile>         ; depura detecções positivas da autohostlist

 --split-pos=N|-N|marker+N|marker-N      ; lista de posições de divisão separadas por vírgula
                                          ; marcadores: method,host,endhost,sld,endsld,midsld,sniext
 --split-any-protocol                    ; divide não apenas http e TLS
 --disorder[=http|tls]                   ; ao dividir, simula o envio do segundo fragmento primeiro
 --oob[=http|tls]                        ; ao dividir, envia byte out-of-band. padrão é HEX 0x00.
 --oob-data=<char>|0xHEX                 ; substitui o byte OOB padrão 0x00.
 --hostcase                              ; altera Host: => host:
 --hostspell                             ; grafia exata do cabeçalho "Host". deve ter 4 caracteres. padrão é "host"
 --hostdot                               ; adiciona "." depois do nome em Host:
 --hosttab                               ; adiciona tab depois do nome em Host:
 --hostnospace                           ; remove o espaço depois de Host:
 --hostpad=<bytes>                       ; adiciona cabeçalhos de preenchimento dummy antes de Host:
 --domcase                               ; mistura capitalização do domínio depois de Host: assim : TeSt.cOm
 --methodspace                           ; adiciona espaço extra depois do método
 --methodeol                             ; adiciona fim de linha antes do método
 --unixeol                               ; substitui 0D0A por 0A
 --tlsrec=N|-N|marker+N|marker-N         ; cria 2 registros TLS. divide na parte lógica especificada. não divide se SNI não estiver presente.
 --tlsrec-pos=<pos>                      ; cria 2 registros TLS. divide na posição especificada
 --mss=<int>                             ; define MSS do cliente. força o servidor a dividir mensagens, mas reduz muito a velocidade !
 --tamper-start=[n]<pos>                 ; começa a modificar apenas a partir da posição de fluxo de saída especificada. posição em bytes ou número de bloco ('n'). padrão 0.
 --tamper-cutoff=[n]<pos>                ; não modificar mais após a posição de fluxo de saída especificada. posição em bytes ou número de bloco ('n'). padrão ilimitado.
 --daemon                                ; executa como daemon
 --pidfile=<filename>                    ; grava o pid em arquivo
 --user=<username>                       ; remove privilégios de root
 --uid=uid[:gid1,gid2,...]               ; remove privilégios de root
```

### Segmentação TCP no tpws

O **tpws**, como o **nfqws**, suporta múltiplas divisões. Os [marcadores de divisão](#segmentação-tcp) são especificados no parâmetro `--split-pos`.

No nível de soquete não há forma garantida de forçar o sistema operacional a enviar pedaços de dados em pacotes separados. O sistema operacional tem um buffer de envio para cada soquete. Se a opção de soquete `TCP_NODELAY` estiver habilitada e o buffer de envio estiver vazio, o sistema operacional provavelmente enviará os dados imediatamente. Se o buffer não estiver vazio, o sistema operacional os combinará com novos dados e enviará em um único pacote, se possível.

Na prática, fora de transmissões massivas, costuma ser suficiente habilitar `TCP_NODELAY` e usar chamadas `send()` separadas para forçar a segmentação TCP personalizada. Mas se houver muitos segmentos, o Linux pode combinar alguns pedaços e quebrar o comportamento desejado. BSD e Windows são mais previsíveis nesse caso. Por isso não é recomendado usar muitas divisões. Testes revelaram que 8+ podem se tornar problemáticos.

Desde o kernel Linux 4.6 o **tpws** consegue reconhecer falhas de segmentação TCP e avisar sobre elas. `--fix-seg` pode corrigir falhas de segmentação ao custo de alguma lentidão. Ele aguarda alguns milissegundos até que todos os dados anteriores sejam enviados. Isso quebra o modelo de processamento assíncrono e deixa mais lenta toda conexão que passa pelo **tpws**. Portanto, não é recomendado em sistemas muito carregados. Mas pode ser um compromisso para sistemas domésticos.

Se você tentar dividir uma transmissão massiva com a opção `--split-any-protocol`, isso com certeza causará falhas massivas de segmentação. Não faça isso sem os limitadores `--tamper-start` e `--tamper-cutoff`.

O **tpws** trabalha no nível de soquete e recebe de uma vez requisições longas (TLS com kyber) que normalmente exigiriam vários pacotes TCP. Ele modifica o bloco recebido inteiro sem saber quantos pacotes serão necessários. O sistema operacional fará segmentação adicional para respeitar o MTU.

`--disorder` envia cada pacote ímpar com TTL=1. O servidor recebe os pacotes pares rapidamente. Depois o sistema operacional do cliente retransmite os pacotes ímpares com TTL normal e o servidor os recebe. No caso de 6 segmentos, servidor e DPI os verão nesta ordem : `2 4 6 1 3 5`. Essa forma de desordem causa alguns atrasos. O timeout padrão de retransmissão no Linux é 200 ms.

`--oob` envia um byte out-of-band no final do primeiro segmento dividido.

`--oob` e `--disorder` podem ser combinados apenas no Linux. Outros sistemas operacionais não tratam isso corretamente.

### TLSREC

`--tlsrec` permite dividir o TLS ClientHello em 2 registros TLS em um único segmento TCP. Aceita um único marcador de posição.

`--tlsrec` quebra um número significativo de sites. Bibliotecas cripto em servidores costumam aceitar um ClientHello modificado sem problemas, mas middleboxes como CDNs e proteções DDoS — nem sempre. O uso de `--tlsrec` sem filtros é desaconselhado.

### MSS

`--mss` define a opção de soquete TCP_MAXSEG. O cliente define esse valor na opção MSS do pacote SYN.
O servidor responde com seu próprio MSS no pacote SYN,ACK. Normalmente os servidores reduzem o tamanho de seus pacotes, mas ainda não se encaixam no MSS fornecido. Quanto maior o MSS definido pelo cliente, maiores serão os pacotes do servidor.
Se isso for suficiente para dividir o ServerHello do TLS 1.2, pode enganar um DPI que verifica o nome de domínio do certificado.
Esse esquema pode reduzir bastante a velocidade. O filtro hostlist é possível apenas no modo socks se o cliente usar resolução remota (firefox `network.proxy.socks_remote_dns`) ou se `ipcache-hostname` estiver habilitado.
`--mss` não é necessário para TLS1.3. Se TLS1.3 pode ser negociado, o MSS só piora as coisas. Use apenas se nada melhor estiver disponível. Funciona apenas no Linux, não em BSD ou MacOS.

### Outras opções de modificação

`--hostpad=<bytes>` adiciona cabeçalhos de preenchimento antes de `Host:` com o número de bytes especificado. Se `<bytes>` for muito grande, os cabeçalhos são divididos em blocos de 2K. Preenchimento acima de 64K não é suportado e não é aceito por servidores http.

É útil contra DPIs com estado que remontam apenas uma quantidade limitada de dados. Aumente `<bytes>` até o site funcionar. Se o `<bytes>` mínimo que funciona estiver próximo do MTU, provavelmente o DPI não está remontando os pacotes. Nesse caso, é melhor usar a divisão comum em vez de `--hostpad`.

### Opções suplementares

O **tpws** pode vincular-se a múltiplas interfaces e endereços IP (até 32).

O número da porta é sempre o mesmo.

Os parâmetros `--bind-iface*` e `--bind-addr` criam um novo vínculo.

Os demais parâmetros `--bind-*` estão relacionados ao último vínculo.

Seleção de modo para ipv6 link local (`fe80::/8`) :

```
--bind-iface6 --bind-linklocal=no : primeiro seleciona endereço privado fc00::/7, depois endereço global
--bind-iface6 --bind-linklocal=unwanted : primeiro seleciona endereço privado fc00::/7, depois endereço global, depois LL
--bind-iface6 --bind-linklocal=prefer : primeiro seleciona LL, depois endereço privado fc00::/7, depois endereço global
--bind-iface6 --bind-linklocal=force : seleciona apenas LL
```

Para vincular a todos os ipv4, especifique `--bind-addr "0.0.0.0"`; para todos os ipv6, `::`.

`--bind-addr=""` — significa vincular a todos os ipv4 e ipv6.

Se nenhum vínculo for especificado, cria-se por padrão um vínculo a todos os endereços ipv4 e ipv6.

Para vincular a um endereço link local específico, faça : `--bind-iface6=fe80::aaaa:bbbb:cccc:dddd%iface-name`

Os parâmetros `--bind-wait*` podem ajudar em situações em que você precisa obter o IP da interface, mas ele ainda não está lá, a interface não subiu ou não está configurada.

Em diferentes sistemas, eventos de ifup são capturados de formas diferentes e não garantem que a interface já tenha recebido um endereço IP de determinado tipo.

No caso geral, não existe um mecanismo único para esperar por um evento do tipo "endereço link local apareceu na interface X".

Para vincular a um ip específico quando a interface pode ainda não estar configurada, faça : `--bind-addr=192.168.5.3 --bind-wait-ip=20`

É possível vincular a qualquer endereço inexistente no modo transparente, mas no modo socks o endereço deve existir.

No modo de proxy socks não são necessários privilégios extras do sistema. Conexões aos IPs locais do sistema onde o **tpws** roda são proibidas.
O tpws suporta resolução DNS remota (curl : `--socks5-hostname`  firefox : `socks_remote_dns=true`), mas a faz em modo bloqueante.

O **tpws** usa soquetes assíncronos para todas as atividades. Nomes de domínio são resolvidos em um pool multithread.
A resolução não congela outras conexões. Mas se houver muitas requisições, atrasos na resolução podem aumentar.
O número de threads de resolução é escolhido automaticamente de forma proporcional a `--maxconn` e pode ser substituído com `--resolver-threads`.
Para desabilitar a resolução de hostname, use a opção `--no-resolve`.

### Múltiplas estratégias

O **tpws**, como o **nfqws**, suporta múltiplas estratégias. Funcionam de forma muito semelhante às do **nfqws**, com diferenças mínimas.
`filter-udp` está ausente porque o **tpws** não suporta udp. Métodos de dessincronização de fase 0 (`--mss`) podem funcionar com hostlist em modos socks com resolução remota de hostname.
Este é o ponto em que você precisa planejar os perfis com cuidado. Se você usar `--mss` e filtros hostlist, o comportamento pode ser diferente dependendo de a resolução remota estar habilitada ou não.
Use `--mss` tanto no perfil com hostlist quanto no perfil sem hostlist.
Use `curl --socks5` e `curl --socks5-hostname` para emitir os dois tipos de consultas de proxy.
Veja a saída de `--debug` para testar sua configuração.

### IPTABLES para tpws

Use as seguintes regras para redirecionar conexões TCP ao tpws :
```
iptables -t nat -I OUTPUT -o <wan_interface> -p tcp --dport 80 -m owner ! --uid-owner tpws -j DNAT --to 127.0.0.127:988
iptables -t nat -I PREROUTING -i <lan_interface> -p tcp --dport 80 -j DNAT --to 127.0.0.127:988
```

A primeira regra redireciona o tráfego originado no próprio sistema; a segunda redireciona o tráfego de passagem.

DNAT para localhost funciona apenas na cadeia **OUTPUT** e não funciona na cadeia **PREROUTING** sem definir este sysctl :

`sysctl -w net.ipv4.conf.<lan_interface>.route_localnet=1`

Também é possível usar `-j REDIRECT --to-port 988` em vez de DNAT, mas nesse caso o proxy transparente precisa escutar em todos os endereços IP ou em um endereço da interface LAN. Não é bom escutar em todos os IPs, e obter um IP específico em um script shell não é trivial. `route_localnet` tem seu próprio impacto de segurança se não for protegido por regras adicionais. Você abre a sub-rede `127.0.0.0/8` para a rede.

Veja como abrir apenas o endereço `127.0.0.127` :
```
iptables -A INPUT ! -i lo -d 127.0.0.127 -j ACCEPT
iptables -A INPUT ! -i lo -d 127.0.0.0/8 -j DROP
```

O filtro owner é necessário para evitar loops de redirecionamento. O **tpws** deve ser executado com o parâmetro `--user tpws`.

O ip6tables funciona quase da mesma forma, com pequenas diferenças. Endereços ipv6 devem ser colocados entre colchetes :
```
ip6tables -t nat -I OUTPUT -o <wan_interface> -p tcp --dport 80 -m owner ! --uid-owner tpws -j DNAT --to [::1]:988
```

Não existe `route_localnet` para ipv6. DNAT para localhost (`::1`) é possível apenas na cadeia **OUTPUT**. Na cadeia **PREROUTING**, o DNAT é possível para qualquer endereço global ou endereço link local da interface de onde o pacote veio.

### NFTABLES para tpws

Esquema base de nftables :
```
IFACE_WAN=wan
IFACE_LAN=br-lan

sysctl -w net.ipv4.conf.$IFACE_LAN.route_localnet=1

nft create table inet ztest

nft create chain inet ztest localnet_protect
nft add rule inet ztest localnet_protect ip daddr 127.0.0.127 return
nft add rule inet ztest localnet_protect ip daddr 127.0.0.0/8 drop
nft create chain inet ztest input "{type filter hook input priority filter - 1;}"
nft add rule inet ztest input iif != "lo" jump localnet_protect

nft create chain inet ztest dnat_output "{type nat hook output priority dstnat;}"
nft add rule inet ztest dnat_output meta skuid != tpws oifname $IFACE_WAN tcp dport { 80, 443 } dnat ip to 127.0.0.127:988
nft create chain inet ztest dnat_pre "{type nat hook prerouting priority dstnat;}"
nft add rule inet ztest dnat_pre meta iifname $IFACE_LAN tcp dport { 80, 443 } dnat ip to 127.0.0.127:988
```

Excluir a nftable :
```
nft delete table inet ztest
```


## Formas de obter uma lista de IPs bloqueados

O nftables não consegue trabalhar com ipsets. Os conjuntos nativos do nf exigem muita RAM para carregar listas grandes de IPs com sub-redes e intervalos.
Se você estiver em um sistema com pouca RAM e precisar de listas grandes, pode ser necessário voltar a iptables+ipset.

1. Informe os domínios bloqueados em `ipset/zapret-hosts-user.txt` e execute `ipset/get_user.sh`
Como resultado, você obtém `ipset/zapret-ip-user.txt` com os endereços IP.

2. `ipset/get_reestr_*.sh`. Específico da Rússia.

3. `ipset/get_antifilter_*.sh`. Específico da Rússia.

4. `ipset/get_config.sh`. Este script chama o que estiver escrito na variável GETLIST do arquivo de configuração.

Se a variável não estiver definida, apenas as listas para os ipsets nozapret/nozapret6 são resolvidas.

Portanto, se você não é russo, a única forma é adicionar manualmente os domínios bloqueados.
Ou escrever seu próprio `ipset/get_iran_blocklist.sh`, se você souber onde baixar essa lista.

Em roteadores, não é recomendado chamar esses scripts mais de uma vez a cada 2 dias, para minimizar escritas na memória flash.

`ipset/create_ipset.sh` executa uma atualização forçada do ipset.
Com o parâmetro `no-update`, `create_ipset.sh` cria o ipset, mas só o preenche se ele tiver sido realmente criado.

Isso é útil quando várias chamadas consecutivas são possíveis, para evitar desperdiçar tempo de CPU refazendo o mesmo trabalho.

O carregamento de ipset consome recursos. É uma boa ideia chamar create_ipset sem o parâmetro `no-update`

apenas uma vez a cada vários dias. Use a opção `no-update` nos demais casos.

Os scripts ipset chamam automaticamente o utilitário ip2net.
O ip2net ajuda a reduzir o tamanho da lista de IPs combinando IPs em sub-redes. Também remove IPs inválidos da lista.
As listas armazenadas já foram processadas pelo ip2net. Estão sem erros e prontas para carregamento.

`create_ipset.sh` suporta carregamento de listas de IPs de arquivos gzip. Primeiro procura o arquivo com extensão ".gz",
como `zapret-ip.txt.gz`; se não encontrar, usa o nome original `zapret-ip.txt`.

Portanto, seu próprio get_iran_blockslist.sh pode usar a função "zz" para gerar o gz. Estude como os outros scripts russos `get_XXX.sh` funcionam.

Comprimir em gzip ajuda a economizar muito do precioso espaço em flash em sistemas embarcados.

Listas de usuário não são comprimidas porque não devem ser muito grandes.

Você pode adicionar uma lista de domínios a `ipset/zapret-hosts-user-ipban.txt`. Seus endereços IP serão colocados
em um ipset separado chamado "ipban". Ele pode ser usado para rotear conexões ao proxy transparente "redsocks" ou à VPN.

IPV6 : se ipv6 estiver habilitado, arquivos txt adicionais são criados com o mesmo nome, mas com um "6" no final antes da extensão.

`zapret-ip.txt` => `zapret-ip6.txt`

Os ipsets zapret6 e ipban6 são criados.

SISTEMA DE EXCLUSÃO DE IP. Todos os scripts resolvem o arquivo `zapret-hosts-user-exclude.txt`, criando `zapret-ip-exclude.txt` e `zapret-ip-exclude6.txt`.

Eles são a fonte dos ipsets nozapret/nozapret6. Todas as regras criadas pelos scripts de inicialização são criadas com esses ipsets em mente.
Os IPs colocados neles não participam do processo.
zapret-hosts-user-exclude.txt pode conter domínios, endereços ipv4 e ipv6 ou sub-redes.

FreeBSD. Os scripts `ipset/*.sh` também funcionam no FreeBSD. Em vez de ipset, criam tabelas de lookup do ipfw com os mesmos nomes do Linux.
Tabelas ipfw podem armazenar endereços e sub-redes ipv4 e ipv6. Não há separação entre 4 e 6.

O parâmetro de configuração LISTS_RELOAD define um comando personalizado de recarregamento de listas.
É útil em sistemas BSD com PF.
LISTS_RELOAD=- desabilita o recarregamento do backend de lista de IPs.

## Filtragem por nome de domínio

Uma alternativa ao ipset é usar **tpws** ou **nfqws** com uma ou mais listas de domínios.
Tanto **tpws** quanto **nfqws** aceitam qualquer número de listas de inclusão (`--hostlist`) e exclusão (`--hostlist-exclude`).
Todas as listas do mesmo tipo são combinadas internamente, restando apenas 2 listas : inclusão e exclusão.

A lista de exclusão é verificada primeiro. O engano é cancelado se o domínio pertencer à lista de exclusão.
Se houver uma lista de inclusão e o domínio não pertencer a ela, o engano também é cancelado.
Lista vazia significa lista ausente. Caso contrário, o engano continua.

O sistema de inicialização procura 2 listas de inclusão :

`ipset/zapret-hosts-users.txt.gz` ou `ipset/zapret-hosts-users.txt`

`ipset/zapret-hosts.txt.gz` ou `ipset/zapret-hosts.txt`

e 1 lista de exclusão

`ipset/zapret-hosts-users-exclude.txt.gz` ou `ipset/zapret-hosts-users-exclude.txt`

Se `MODE_FILTER=hostlist`, todas as listas presentes são passadas ao **nfqws** ou **tpws**.
Se todas as listas de inclusão estiverem vazias, funciona como se não existissem listas de inclusão.
Se você precisar do modo "todos exceto", não precisa apagar zapret-hosts-users.txt. Basta deixá-lo vazio.

Subdomínios são aplicados automaticamente. Por exemplo, "ru" na lista afeta "\*.ru".
O prefixo `^` desabilita a correspondência de subdomínio.

**tpws** e **nfqws** recarregam as listas automaticamente se o horário de modificação ou o tamanho do arquivo mudar.
O sinal HUP força o recarregamento de todas as listas.

Ao filtrar por nome de domínio, os daemons devem rodar sem filtragem por ipset.
Ao usar listas grandes de órgãos reguladores, calcule a quantidade de RAM no roteador !

## Modo **autohostlist**

Este modo analisa tanto requisições do cliente quanto respostas do servidor.
Se um host não estiver em nenhuma lista e ocorrer uma situação semelhante a um bloqueio, o host é adicionado automaticamente a uma lista especial, tanto em memória quanto em arquivo.
Use a hostlist de exclusão para impedir o disparo da autohostlist.
Se isso acontecer — remova o registro indesejado do arquivo.

No caso do nfqws, é necessário redirecionar tanto o tráfego de entrada quanto o de saída para a fila.
É fortemente recomendado usar o filtro connbytes, ou o nfqws processará gigabytes de tráfego de entrada.
Pelo mesmo motivo, não é recomendado usar o modo autohostlist em BSDs. BSDs não suportam connbytes ou mecanismo similar.

**nfqws** e **tpws** detectam as seguintes situações :
1) [nfqws] Múltiplas retransmissões da primeira requisição dentro de uma sessão TCP que possui host.
2) [nfqws,tpws] RST em resposta à primeira requisição.
3) [nfqws,tpws] Redirecionamento HTTP em resposta à primeira requisição http com domínio de segundo nível diferente do original.
4) [tpws] Cliente fecha a conexão após a primeira requisição sem resposta do servidor (sem resposta, timeout).

Para minimizar falsos positivos existe um contador de falhas. Se, em determinado intervalo, ocorrerem mais que o número especificado de falhas,
o host é adicionado à lista. Então a estratégia de burla do DPI começa a ser aplicada imediatamente.

Para o usuário, o modo autohostlist se parece com isto.
Quando o usuário visita um site bloqueado pela primeira vez, vê a página de bloqueio, a conexão é redefinida ou o navegador trava até o timeout e então exibe um erro.
O usuário pressiona F5 várias vezes, fazendo o navegador tentar de novo.
Após algumas tentativas, o site abre e da próxima vez funciona como esperado.

Com o modo autohostlist é possível usar estratégias de burla que quebram muitos sites.
Se um site não se comporta como bloqueado, nenhum engano é aplicado.
Caso contrário, não há nada a perder.

No entanto, falsos positivos ainda podem ocorrer se o site-alvo estiver se comportando de forma anormal
(pode ser por um ataque DDoS ou falha do servidor). Se isso acontecer, a estratégia de burla
pode começar a quebrar o site. Essa situação só pode ser controlada manualmente.
Remova o domínio indesejado do arquivo autohostlist.
Use a hostlist de exclusão para impedir novas adições automáticas.

É possível usar uma autohostlist com múltiplos processos. Todos os processos verificam o horário de modificação do arquivo.
Se um processo modificar a autohostlist, todos os outros a relerão automaticamente.
Todos os processos devem rodar com o mesmo uid.

Se os scripts do zapret forem usados, a autohostlist é `ipset/zapret-hosts-auto.txt`
e a lista de exclusão é `ipset/zapret-hosts-user-exclude.txt`. O modo autohostlist
inclui o modo hostlist. Você pode usar `ipset/zapret-hosts-user.txt`.


## Escolhendo parâmetros

O arquivo `/opt/zapret/config` é usado por vários componentes do sistema e contém as configurações básicas.
Ele precisa ser visualizado e editado, se necessário.

Qual tipo de firewall usar em sistemas Linux : `nftables` ou `iptables`.
Em sistemas tradicionais, `nftables` é selecionado por padrão se `nft` estiver instalado.
No OpenWRT, por padrão, `nftables` é selecionado em sistemas baseados em `firewall4`.

`FWTYPE=iptables`

Com `nftables`, o esquema pós-NAT é usado por padrão. Ele permite mais ataques ao DPI em tráfego encaminhado.
É possível usar o esquema pré-NAT, parecido com iptables. O **nfqws** verá os IPs de origem dos clientes e os exibirá nos logs.

`#POSTNAT=0`

Existem 3 opções padrão configuradas separada e independentemente : `tpws-socks`, **tpws** e **nfqws**.
Podem ser usadas sozinhas ou combinadas. Scripts personalizados em `init.d/{sysv,openwrt,macos}/custom.d` são sempre aplicados.

`tpws-socks` exige configuração dos parâmetros do daemon, mas não exige interceptação de tráfego.
As outras opções padrão também exigem interceptação de tráfego.
Cada opção padrão inicia uma única instância do daemon. Diferenças de estratégia são gerenciadas com o esquema de múltiplos perfis.
A regra principal para interceptação é "intercepte o mínimo necessário". Todo o resto apenas desperdiça recursos de CPU e deixa a conexão mais lenta.

A opção `--ipset` é proibida intencionalmente para impedir a filtragem fácil, mas ineficiente, em modo usuário.
Use ipsets do kernel. Isso pode exigir scripts personalizados.

Para usar as hostlists atualizáveis padrão do diretório `ipset`, use o marcador `<HOSTLIST>`. Ele é substituído automaticamente
pelos parâmetros de hostlist se a variável `MODE_FILTER` habilitar hostlists e é removido caso contrário.
As hostlists padrão devem estar nas estratégias finais (fallback), fechando grupos de parâmetros de filtro.
Não use `<HOSTLIST>` em perfis altamente especializados. Use seu próprio filtro ou hostlist(s).
O marcador `<HOSTLIST_NOAUTO>` usa a autohostlist padrão como uma hostlist comum, desabilitando assim adições automáticas neste perfil.
Se qualquer outro perfil adicionar algo, este perfil aceita a mudança automaticamente.

Alterar o bit de mark de prevenção de loops :

`DESYNC_MARK=0x40000000`

Alterar o bit de mark do esquema postnat :

`DESYNC_MARK_POSTNAT=0x20000000`

Se descomentado, passa ao zapret apenas os pacotes marcados com este bit :

`#FILTER_MARK=0x10000000`

O bit deve ser definido em suas próprias regras.
* iptables — em mangle PREROUTING e mangle OUTPUT, antes das regras do zapret (iptables -I _depois_ da aplicação das regras do zapret).
* nftables — nos hooks output e prerouting com prioridade -102 ou menor.

Os critérios de mark podem ser quaisquer. Por exemplo, IP de origem ou nome da interface de origem.

Chaveamento do modo proxy socks do **tpws** :

`TPWS_SOCKS_ENABLE=0`

Porta TCP de escuta para o modo proxy do **tpws** :

`TPPORT_SOCKS=987`

Parâmetros do **tpws** no modo socks :

```
TPWS_SOCKS_OPT="
--filter-tcp=80 --methodeol <HOSTLIST> --new
--filter-tcp=443 --split-pos=1,midsld --disorder <HOSTLIST>"
"
```

Chaveamento do modo transparente do **tpws** :

`TPWS_ENABLE=0`

Portas-alvo do **tpws** no modo transparente :

`TPWS_PORTS=80,443`

Parâmetros do **tpws** no modo transparente :

```
TPWS_OPT="
--filter-tcp=80 --methodeol <HOSTLIST> --new
--filter-tcp=443 --split-pos=1,midsld --disorder <HOSTLIST>"
"
```

Chaveamento de habilitação do **nfqws** :

`NFQWS_ENABLE=0`

Portas-alvo do **nfqws** para interceptação limitada por `connbytes`. `connbytes` permite interceptar apenas os pacotes iniciais das conexões.
É uma alternativa mais eficiente, em modo kernel, a `nfqws --dpi-desync-cutoff=nX`.

```
NFQWS_PORTS_TCP=80,443
NFQWS_PORTS_UDP=443
```

Quantos pacotes iniciais devem ser interceptados pelo nfqws em cada direção :

```
NFQWS_TCP_PKT_OUT=$((6+$AUTOHOSTLIST_RETRANS_THRESHOLD))
NFQWS_TCP_PKT_IN=3
NFQWS_UDP_PKT_OUT=$((6+$AUTOHOSTLIST_RETRANS_THRESHOLD))
NFQWS_UDP_PKT_IN=0
```

Existe um tipo de tráfego que exige a interceptação de todo o fluxo de saída.
Normalmente é suporte a keepalives de http plano e DPI sem estado.
Esse modo de interceptação aumenta bastante o uso de CPU. Use com cuidado e apenas quando necessário.
Aqui você especifica os números de porta para interceptação ilimitada.
É aconselhável também remover essas portas da lista de interceptação limitada por `connbytes`.

```
#NFQWS_PORTS_TCP_KEEPALIVE=80
#NFQWS_PORTS_UDP_KEEPALIVE=
```

Parâmetros do **nfqws** :

```
NFQWS_OPT="
--filter-tcp=80 --dpi-desync=fake,multisplit --dpi-desync-split-pos=method+2 --dpi-desync-fooling=md5sig <HOSTLIST> --new
--filter-tcp=443 --dpi-desync=fake,multidisorder --dpi-desync-split-pos=1,midsld --dpi-desync-fooling=badseq,md5sig <HOSTLIST> --new
--filter-udp=443 --dpi-desync=fake --dpi-desync-repeats=6 <HOSTLIST_NOAUTO>
"
```


Modo de filtragem de hosts :
```
none - aplica o engano a todos os hosts
ipset - limita o engano a hosts do ipset zapret/zapret6
hostlist - limita o engano a hosts da hostlist
autohostlist - modo hostlist + detecção automática de bloqueios
```

`MODE_FILTER=none`


controle de flow offloading (se suportado) :

```
donttouch : desabilita a configuração de flow offloading do sistema se o modo selecionado for incompatível com ela; caso contrário, não mexe nela e não configura offloading seletivo
none : sempre desabilita a configuração de flow offloading do sistema e não configura offloading seletivo
software : sempre desabilita a configuração de flow offloading do sistema e configura offloading seletivo por software
hardware : sempre desabilita a configuração de flow offloading do sistema e configura offloading seletivo por hardware
```

`FLOWOFFLOAD=donttouch`

O parâmetro GETLIST informa ao instalador install_easy.sh qual script chamar
para atualizar a lista de IPs ou hosts bloqueados.
Ele é chamado via `get_config.sh` a partir de tarefas agendadas (crontab ou timer systemd).
Coloque aqui o nome do script que você usará para atualizar as listas.
Se não for usar, o parâmetro deve ficar comentado.

Você pode desabilitar individualmente ipv4 ou ipv6. Se o parâmetro estiver comentado ou diferente de "1",
o uso do protocolo é permitido.

```
#DISABLE_IPV4=1
DISABLE_IPV6=1
```

O número de threads para o resolvedor DNS multithread do mdig (1..100).
Quanto mais, mais rápido, mas seu servidor DNS vai se ofender com tanta insistência ?

`MDIG_THREADS=30`

diretório temporário. Usado pelos scripts ipset/*.sh para processar listas grandes.
/tmp por padrão. Pode ser reatribuído se /tmp for tmpfs e a RAM for baixa.
TMPDIR=/opt/zapret/tmp

opções de ipset e nfset :

```
SET_MAXELEM=262144
IPSET_OPT="hashsize 262144 maxelem 2097152
```

O kernel aumenta automaticamente o hashsize se o ipset for grande demais para o hashsize atual.
Esse procedimento exige realocação interna e pode precisar de memória adicional.
Em sistemas com pouca RAM pode causar erros.
Não use um hashsize alto demais. Dessa forma você desperdiça sua RAM. E não use um hashsize baixo demais para evitar realocações.

opções do ip2net, separadas para ipv4 e ipv6 :

```
IP2NET_OPT4="--prefix-length=22-30 --v4-threshold=3/4"
IP2NET_OPT6="--prefix-length=56-64 --v6-threshold=5"
```

ajuste do modo autohostlist :

```
AUTOHOSTLIST_RETRANS_THRESHOLD=3
AUTOHOSTLIST_FAIL_THRESHOLD=2
AUTOHOSTLIST_FAIL_TIME=60
AUTOHOSTLIST_DEBUG=0
```

Habilita compressão gzip para listas grandes. Usado pelos scripts ipset/*.sh.

`GZIP_LISTS=1`

Comando para recarregar listas de ip/host após atualização.
Comente ou deixe vazio para seleção automática de backend : ipset ou ipfw, se presentes.
Em sistemas BSD com PF não ocorre recarregamento automático. Você deve fornecer seu próprio comando.
Versões mais novas do FreeBSD suportam recarregamento apenas de tabelas : `pfctl -Tl -f /etc/pf.conf`
Defina como "-" para desabilitar o recarregamento.

`LISTS_RELOAD="pfctl -f /etc/pf.conf"`

No OpenWRT existe a rede padrão `lan`. Apenas o tráfego vindo desta rede é redirecionado ao tpws por padrão.
Para substituir esse comportamento, defina a seguinte variável :

`OPENWRT_LAN="lan lan2 lan3"`

No OpenWRT, interfaces wan são aquelas com rota padrão. Separadamente para ipv4 e ipv6.
Isso pode ser redefinido :
```
OPENWRT_WAN4="wan4 vpn"
OPENWRT_WAN6="wan6 vpn6"
```

O parâmetro `INIT_APPLY_FW=1` habilita o script de inicialização a aplicar regras iptables de forma independente.
Com outros valores, ou se o parâmetro estiver comentado, as regras não serão aplicadas.
Isso é útil se você tem um sistema de gerenciamento de firewall, nas configurações do qual deve atrelar as regras.
Não se aplica ao `OpenWRT` se usado com `firewall3+iptables`.

`FILTER_TTL_EXPIRED_ICMP=1` bloqueia mensagens icmp time exceeded em resposta a conexões tratadas pelo nfqws.
O Linux fecha o soquete se receber esse icmp em resposta a um pacote SYN. Existe mecanismo similar para soquetes de datagrama.
É melhor desabilitar isso se você não espera problemas causados por icmp.

As seguintes configurações não são relevantes para o openwrt :

Se seu sistema funciona como roteador, você precisa informar os nomes das interfaces interna e externa:
```
IFACE_LAN=eth0
IFACE_WAN=eth1
IFACE_WAN6="henet ipsec0"
```
Múltiplas interfaces são separadas por espaço. Se IFACE_WAN6 for omitido, o valor de IFACE_WAN é usado.

IMPORTANTE: configurar roteamento, masquerade etc. não é tarefa do zapret.
Apenas os modos que interceptam tráfego de trânsito são habilitados.
É possível especificar múltiplas interfaces assim : `IFACE_LAN="eth0 eth1 eth2"`


## Integrando ao sistema de controle de firewall ou ao seu sistema de inicialização

Se você usa algum tipo de sistema de gerenciamento de firewall, ele pode entrar em conflito com um script de inicialização existente.
Ao reaplicar as regras, ele pode quebrar as configurações de iptables do zapret.
Nesse caso, as regras de iptables devem ser integradas ao seu firewall separadamente da execução do tpws ou nfqws.

As seguintes chamadas permitem aplicar ou remover regras iptables separadamente :

```
 /opt/zapret/init.d/sysv/zapret start_fw
 /opt/zapret/init.d/sysv/zapret stop_fw
 /opt/zapret/init.d/sysv/zapret restart_fw
```

E você pode iniciar ou parar os daemons separadamente do firewall :

```
 /opt/zapret/init.d/sysv/zapret start_daemons
 /opt/zapret/init.d/sysv/zapret stop_daemons
 /opt/zapret/init.d/sysv/zapret restart_daemons
```

O nftables praticamente elimina conflitos entre sistemas de controle de firewall porque permite tabelas e hooks netfilter separados. A tabela nf `zapret` é usada para os fins do zapret.
Se seu sistema não mexer nela, provavelmente tudo ficará bem.

Existem algumas chamadas adicionais exclusivas de nftables :

Consulta os conjuntos de interfaces `lanif`, `wanif`, `wanif6` e `flow table`.
```
 /opt/zapret/init.d/sysv/zapret list_ifsets
```

Renova os conjuntos de interfaces `lanif`, `wanif`, `wanif6` e `flow table`.
Obtidos das variáveis de configuração `IFACE_LAN`, `IFACE_WAN` em sistemas Linux tradicionais.
Selecionados automaticamente no `OpenWRT`. `lanif` pode ser estendido com a variável de configuração `OPENWRT_LAN`.
```
 /opt/zapret/init.d/sysv/zapret reload_ifsets
```

Chama `nft -t list table inet zapret`.
```
 /opt/zapret/init.d/sysv/zapret list_table
```

Também é possível acoplar seu script a qualquer estágio do processamento do firewall do zapret.
As seguintes configurações estão disponíveis no arquivo de configuração do zapret :

```
INIT_FW_PRE_UP_HOOK="/etc/firewall.zapret.hook.pre_up"
INIT_FW_POST_UP_HOOK="/etc/firewall.zapret.hook.post_up"
INIT_FW_PRE_DOWN_HOOK="/etc/firewall.zapret.hook.pre_down"
INIT_FW_POST_DOWN_HOOK="/etc/firewall.zapret.hook.post_down"
```

Hooks são extremamente úteis se você precisa que conjuntos nftables sejam populados pelos scripts do zapret.
Conjuntos nf podem pertencer a apenas uma tabela. Você precisa escrever a regra lá e sincronizá-la com os scripts do zapret.

## Instalação

### Verificando o ISP

Antes de executar o zapret você precisa descobrir uma estratégia de burla que funcione.
O `blockcheck.sh` automatiza esse processo. Ele primeiro verifica o DNS e então testa várias estratégias até encontrar as que funcionam.
Note que a verificação de DNS é voltada basicamente à Rússia. Ela verifica vários domínios bloqueados na Rússia predefinidos e
compara as respostas do DNS do sistema com as respostas de DNS públicos. Como o ISP pode bloquear DNS público ou redirecionar qualquer consulta DNS
para seus servidores, o `blockcheck.sh` também verifica se todas as respostas retornadas são únicas. Normalmente, se o DNS está bloqueado,
o ISP retorna um único IP para todos os domínios bloqueados para te redirecionar à página "acesso negado".
Servidores DoH são usados automaticamente se detecção de falsificação de DNS for identificada.
O `blockcheck.sh` funciona em todos os sistemas suportados pelo `zapret`.

### sistema Linux desktop

A instalação simples funciona na maioria das distribuições Linux modernas com systemd ou openrc, OpenWRT e MacOS.
Execute `install_easy.sh` e responda às perguntas.

### OpenWRT

O `install_easy.sh` funciona no OpenWRT, mas há desafios adicionais.
Eles estão relacionados principalmente ao possível espaço livre em flash.
A instalação simples não funcionará se não houver espaço para instalar a si mesmo e os pacotes necessários do repositório.

Outro desafio é levar o zapret ao roteador. Você pode baixar o zip do github e usá-lo.
Instale openssh-sftp-server e unzip no OpenWRT e use sftp para transferir o arquivo.
Também não é difícil usar 'nc' (netcat) para transferir arquivos.

A melhor forma de começar é colocar o diretório zapret em `/tmp` e executar `/tmp/zapret/install_easy.sh` a partir de lá.
Após a instalação, remova `/tmp/zapret` para liberar RAM.

O mínimo absoluto para OpenWRT é 64/8 de sistema, 64/16 é confortável, 128/extroot é recomendado.

Para OpenWRT com pouco armazenamento, veja `init.d/openwrt-minimal`.

### Android

Não é possível usar **nfqws** e **tpws** em modo de proxy transparente sem privilégios de root. Sem root, o **tpws** pode rodar no modo `--socks`.

O Android tem NFQUEUE e o **nfqws** deve funcionar.

Não há suporte a `ipset` a menos que você rode um kernel customizado. No caso comum, a tarefa de fazer `ipset` funcionar no Android varia de "não fácil" a "quase impossível", a menos que você encontre uma imagem de kernel funcional para seu dispositivo.

Embora os binários Linux funcionem, é recomendado usar os binários específicos para Android. Eles não têm problemas com nomes de usuário, hora local, DNS etc.
É recomendado usar gid 3003 (AID_INET); caso contrário, o **tpws** não terá acesso à internet.

Exemplo : `--uid 1:3003`

Em iptables use : `! --uid-owner 1` em vez de `! --uid-owner tpws`.

O **nfqws** deve ser executado com `--uid 1`. Caso contrário, em alguns dispositivos ou firmwares o kernel pode travar parcialmente. Parece que processos com certos uids podem ser suspensos. Com um driver de modem celular chinês bugado, isso pode levar o dispositivo a travar.

Escreva seu próprio script shell com iptables e **tpws** e execute-o pelo seu gerenciador de root.
Scripts de autorun ficam aqui :

magisk  : `/data/adb/service.d`

supersu : `/system/su.d`

Como executar o **tpws** em Android sem root.
Você não pode escrever em `/system`, `/data`, nem executar a partir do cartão SD.
O Selinux impede que apps executem binários em `/data/local/tmp`.
Use adb e adb shell.

```
mkdir /data/local/tmp/zapret
adb push tpws /data/local/tmp/zapret
chmod 755 /data/local/tmp/zapret /data/local/tmp/zapret/tpws
chcon u:object_r:system_file:s0 /data/local/tmp/zapret/tpws
```

Agora é possível executar `/data/local/tmp/zapret/tpws` a partir de qualquer app, como o Tasker.

### FreeBSD, OpenBSD, MacOS

veja a [documentação BSD](./bsd.en.md)

### Windows (WSL)

veja a [documentação Windows](./windows.en.md)

### Outros dispositivos

O objetivo do autor não inclui dar suporte fácil ao maior número possível de dispositivos.
Por favor, não peça suporte fácil para firmwares. Isso exige muito trabalho e ter muitos dispositivos. É contraproducente.
Como proprietário do dispositivo, é mais fácil para você e não deve ser muito difícil se o firmware for aberto.
A maioria dos firmwares stock fechados não foi projetada para uso personalizado e, às vezes, ativamente o impedem.
Nesse caso, você precisa invadir o sistema e fazer engenharia reversa. Não é fácil.
Os binários são universais. Podem rodar em praticamente qualquer firmware.
Você precisará de :
  * acesso root ao shell. Um shell sh de verdade, não um console tipo microtik
  * hook de inicialização
  * partição r/w para armazenar os binários e o script de inicialização com permissão de execução (+x)
  * **tpws** pode rodar em quase qualquer lugar, mas **nfqws** exige suporte do kernel a NFQUEUE. Isso falta na maioria dos firmwares.
  * kernels 2.6 muito antigos não são suportados e podem causar erros. Kernels 2.6 mais novos estão OK.
Se os binários travam com segfault (raro, mas acontece em alguns kernels), tente descompactar o upx assim : upx -d tpws.

Primeiro depure seu cenário manualmente. Execute iptables + daemon e verifique se é o que você quer.
Escreva seu próprio script com a mágica do iptables e execute o daemon necessário a partir dele. Coloque-o na inicialização.
Não me pergunte como fazer isso. É diferente para cada firmware e exige estudo.
Encontre o manual ou faça engenharia reversa você mesmo.
Verifique condições de corrida. O firmware pode limpar ou modificar o iptables depois do seu script de inicialização.
Se for esse o caso, execute outro script em segundo plano e adicione algum atraso nele.

## Doações

São bem-vindas aqui :

USDT ERC `0x3d52Ce15B7Be734c53fc9526ECbAB8267b63d66E`

USDT TRC `TEzAAtn4VhndqEaAyuCM78xh5W2gCjwWEo`

BTC  `bc1qhqew3mrvp47uk2vevt5sctp7p2x9m7m5kkchve`

ETH  `0x3d52Ce15B7Be734c53fc9526ECbAB8267b63d66E`
