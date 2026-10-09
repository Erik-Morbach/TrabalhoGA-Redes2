# Respostas: Trabalho do Grau A (Redes)

**Integrantes:** Erik Cruz Morbach(1946271), Lucas Hoffmeister Escopelli(1916768), Vinicius Muller Silveira(1789484)

**/24 do grupo:** 10.1.2.0/24

---

## Etapa 1: matriz

### Q1. A conversa DHCP completa

Captura no modo Simulation, filtro só DHCP, no PC-ENG-1 (LAN-ENG), depois de `ipconfig /release` e `ipconfig /renew`. Evidências: `evidencias/prints/q1_dhcp_dora.png` e `evidencias/prints/q1_pdu_discover.png` (PDU Details do DISCOVER).

Mensagens observadas na LAN do cliente:

| Mensagem | IP de origem | IP de destino | MAC de destino |
| :--- | :--- | :--- | :--- |
| DISCOVER | 0.0.0.0 | 255.255.255.255 | FFFF.FFFF.FFFF |
| OFFER | 10.1.2.65 | 255.255.255.255 | FFFF.FFFF.FFFF |
| REQUEST | 0.0.0.0 | 255.255.255.255 | FFFF.FFFF.FFFF |
| ACK | 10.1.2.65 | 255.255.255.255 | FFFF.FFFF.FFFF |

O endereço oferecido e confirmado é 10.1.2.68 (campo YOUR CLIENT ADDRESS do OFFER e do ACK).

**Por que as mensagens do cliente partem de 0.0.0.0.** No DISCOVER e no REQUEST o PC-ENG-1 ainda não tem endereço IP: o `ipconfig /release` devolveu o endereço anterior e o cliente só passa a ter IP depois do ACK. Como a camada IP exige um endereço de origem, o cliente usa 0.0.0.0 ("este host, nesta rede"). O destino é 255.255.255.255 (broadcast limitado, MAC FFFF.FFFF.FFFF) porque o cliente também não sabe onde está o servidor DHCP: a mensagem precisa alcançar todos os hosts da LAN, e o relay do MTZ-R2 a encaminha. As portas UDP são 68 (cliente) e 67 (servidor).

### Q2. O relay em ação

DISCOVER observado no MTZ-R2 (modo Simulation, filtro DHCP), comparando *Inbound PDU Details* (antes do relay) e *Outbound PDU Details* (depois). Evidência: `evidencias/prints/q2_giaddr.png`.

**O que muda nos IPs.** O broadcast 255.255.255.255 não atravessa roteador, então o MTZ-R2, configurado com `ip helper-address 10.1.2.146` na g0/0, recebe o DISCOVER e o reenvia como um pacote unicast novo: o destino passa de 255.255.255.255 para o IP do servidor (10.1.2.146) e a origem passa de 0.0.0.0 para o IP da interface do roteador que recebeu o pedido (10.1.2.65). Os MACs também mudam, porque o quadro é refeito a cada salto de camada 2.

**Campo que carrega o endereço da sub-rede do cliente.** O campo **GIADDR** (*Gateway/Relay Agent IP Address*, "RELAY AGENT ADDRESS" no Packet Tracer) passa de 0.0.0.0 para 10.1.2.65, o endereço da interface do MTZ-R2 na LAN-ENG (10.1.2.64/27).

**Como o servidor usa o GIADDR.** O IP de origem do pacote (10.1.2.65) e o GIADDR dizem ao servidor em que sub-rede está o cliente, que ele não consegue deduzir pelo broadcast. O SRV-DC compara o GIADDR com as redes dos seus pools (pela máscara de cada um): 10.1.2.65 pertence a 10.1.2.64/27, então responde com o pool da LAN-ENG (gateway 10.1.2.65, a partir de 10.1.2.66, máscara 255.255.255.224). No caminho de volta, o OFFER é enviado em unicast para o GIADDR, e o MTZ-R2 o entrega ao cliente na LAN. Sem o GIADDR, o servidor não saberia escolher entre o pool da LAN-ENG e o da LAN-ADM.

### Q3. E sem o relay?

`no ip helper-address 10.1.2.146` na g0/0 do MTZ-R2, `ipconfig /release` e `ipconfig /renew` no PC-ENG-1, em modo Simulation com filtro DHCP. Evidência: `evidencias/prints/q3_discover_descartado.png`

**Onde o DISCOVER morre.** Na Event List o pacote sai do PC-ENG-1 (0.000), passa pelo switch LAN-ENG (0.001) e chega ao MTZ-R2 (0.002), e **não há nenhum evento depois dele**: o MTZ-R2 não o encaminha. A aba OSI Model do MTZ-R2 (camada 7, DHCP Packet, origem PC-ENG-1, destino 255.255.255.255) descreve o descarte: *"The DHCP server received a DHCP Discover packet. The DHCP server does not have a pool configured for the received port. It drops the packet."* Sem o helper-address, o roteador trata o DISCOVER como destinado a ele mesmo, não tem serviço DHCP nem pool para a g0/0, e o descarta. O PC-ENG-1 fica sem endereço.

**Por que um broadcast não atravessa um roteador.** O DISCOVER é um broadcast limitado (IP 255.255.255.255, MAC FFFF.FFFF.FFFF), que só vale dentro do domínio de broadcast do cliente. Um roteador separa domínios de broadcast: ele encaminha pacotes pelo IP de destino consultando a tabela de rotas, e 255.255.255.255 não é um destino roteável. Por isso o broadcast fica restrito à LAN-ENG. O `ip helper-address` muda isso apenas para os serviços configurados (inclusive DHCP, UDP 67): o roteador captura o broadcast, transforma-o em unicast para o servidor e preenche o GIADDR (Q2). Sem ele, o servidor, que está em outra sub-rede (LAN-SRV), nunca recebe o pedido.


### Q4. DR e BDR onde não se esperava

O `show ip ospf neighbor` do MTZ-R1 está em `evidencias/etapa1.pdf`, seção 1.1

**Colunas.** *Neighbor ID*: Router ID do vizinho (1.1.1.2 é o MTZ-R2, 1.1.1.3 o MTZ-R3). *Pri*: prioridade OSPF do vizinho na eleição (1 em todos, o padrão). *State*: `FULL` indica que a adjacência está completa e as bases de dados (LSDB) estão sincronizadas; depois da barra vem o **papel do vizinho** no enlace, ou seja, `FULL/DR` significa que o vizinho é o DR. *Dead Time*: tempo restante até declarar o vizinho morto se nenhum Hello chegar (reinicia a cada Hello, máx. 40 s). *Address*: IP do vizinho no enlace. *Interface*: interface local por onde ele foi aprendido.

**DR e BDR em estado atual**, de `show ip ospf interface` nos três roteadores:

```
MTZ-R1>show ip ospf interface g0/1
  Internet address is 10.1.2.153/30, Area 0
  Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State BDR, Priority 1
  Designated Router (ID) 1.1.1.2, Interface address 10.1.2.154
  Backup Designated Router (ID) 1.1.1.1, Interface address 10.1.2.153

MTZ-R1>show ip ospf interface g0/2
  Internet address is 10.1.2.157/30, Area 0
  Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State BDR, Priority 1
  Designated Router (ID) 1.1.1.3, Interface address 10.1.2.158
  Backup Designated Router (ID) 1.1.1.1, Interface address 10.1.2.157

MTZ-R2>show ip ospf interface g0/2
  Internet address is 10.1.2.161/30, Area 0
  Process ID 1, Router ID 1.1.1.2, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State BDR, Priority 1
  Designated Router (ID) 1.1.1.3, Interface address 10.1.2.162
  Backup Designated Router (ID) 1.1.1.2, Interface address 10.1.2.161

MTZ-R3>show ip ospf interface g0/2
  Internet address is 10.1.2.162/30, Area 0
  Process ID 1, Router ID 1.1.1.3, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 1.1.1.3, Interface address 10.1.2.162
  Backup Designated Router (ID) 1.1.1.2, Interface address 10.1.2.161
```

| Enlace | Roteadores | DR | BDR |
| :--- | :--- | :--- | :--- |
| ENL1 (10.1.2.152/30) | MTZ-R1 e MTZ-R2 | MTZ-R2 (1.1.1.2) | MTZ-R1 (1.1.1.1) |
| ENL2 (10.1.2.156/30) | MTZ-R1 e MTZ-R3 | MTZ-R3 (1.1.1.3) | MTZ-R1 (1.1.1.1) |
| ENL3 (10.1.2.160/30) | MTZ-R2 e MTZ-R3 | MTZ-R3 (1.1.1.3) | MTZ-R2 (1.1.1.2) |

**Por que um enlace GigabitEthernet com só dois roteadores elege DR e BDR.** O OSPF decide o comportamento pelo *tipo de rede* da interface, não pelo número de roteadores. Uma interface Ethernet/GigabitEthernet tem tipo padrão `BROADCAST` (visto em `Network Type BROADCAST`), que presume que podem existir vários roteadores no segmento. Nesse tipo a eleição de DR/BDR acontece sempre, mesmo com dois roteadores.

**O roteador de maior Router ID venceu em todos os enlaces?** No estado final do `etapa1.pkt`, sim: com prioridade 1 em todos, o desempate é o maior Router ID, e foi o que se viu (MTZ-R2 > MTZ-R1 no ENL1, MTZ-R3 > MTZ-R1 no ENL2, MTZ-R3 > MTZ-R2 no ENL3).

**Como forçar uma nova eleição.** Reiniciar o processo OSPF dos roteadores do enlace, com `clear ip ospf process` (nos dois lados), ou derrubar/reativar a interface, para que os Hellos recomecem sem DR. Para escolher quem vence, ajusta-se antes a prioridade da interface (`ip ospf priority N`, em que o maior valor vence e 0 impede o roteador de ser eleito) e depois reinicia-se o processo, já que mudar a prioridade sozinho não derruba o DR atual.

### Q5. Anatomia de [110/2]

O `show ip route` do MTZ-R2 está em `evidencias/etapa1.pdf`, seção 1.2. A rota para a LAN-SRV (10.1.2.144/29) e seus detalhes:

```
MTZ-R2#show ip route
O       10.1.2.144/29 [110/2] via 10.1.2.153, 01:38:40, GigabitEthernet0/1

MTZ-R2>show ip route 10.1.2.144
Routing entry for 10.1.2.144/29
Known via "ospf 1", distance 110, metric 2, type intra area
  Last update from 10.1.2.153 on GigabitEthernet0/1, 01:38:57 ago
  Routing Descriptor Blocks:
  * 10.1.2.153, from 1.1.1.1, 01:38:57 ago, via GigabitEthernet0/1
      Route metric is 2, traffic share count is 1
```

**Decomposição de [110/2].** O primeiro número, **110**, é a *distância administrativa* do OSPF, que mede a confiabilidade da origem da rota (menor é melhor; conectada = 0, estática = 1, RIP = 120). O segundo, **2**, é a *métrica* OSPF: o custo total até o destino. A rota é aprendida do MTZ-R1 (`from 1.1.1.1`) e sai por g0/1, com próximo salto 10.1.2.153 (MTZ-R1 no ENL1).

**Conta do custo.** O custo OSPF de um caminho é a soma dos custos das interfaces **de saída** de cada roteador no trajeto, incluindo a interface que conecta a rede de destino:

```
MTZ-R2>show ip ospf interface g0/1
  Internet address is 10.1.2.154/30, Area 0
  Process ID 1, Router ID 1.1.1.2, Network Type BROADCAST, Cost: 1

MTZ-R1>show ip ospf interface g0/0
  Internet address is 10.1.2.145/29, Area 0
  Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
```

| Trecho | Interface de saída | Custo |
| :--- | :--- | :--- |
| MTZ-R2 → MTZ-R1 (ENL1) | MTZ-R2 g0/1 | 1 |
| MTZ-R1 → LAN-SRV | MTZ-R1 g0/0 (anuncia 10.1.2.144/29) | 1 |
| **Total** | | **2** |

**Referência de banda de 100 Mb/s.** O custo de uma interface é `custo = banda de referência / banda da interface`, com referência padrão de 100 Mb/s (10^8 b/s) e arredondamento para baixo, com mínimo de 1. Para GigabitEthernet: 100 / 1000 = 0,1, que vira **1**. Para uma FastEthernet hipotética: 100 / 100 = **1**. Portanto uma FastEthernet no lugar de qualquer enlace do triângulo teria exatamente o mesmo custo 1 que a GigabitEthernet. (Para comparação, uma serial de 1,544 Mb/s custa 100 / 1,544 ≈ 64.)

**O que isso revela.** Com a referência padrão, o OSPF não consegue distinguir enlaces acima de 100 Mb/s: Fast, Giga e 10 Giga recebem todos custo 1, e o protocolo trata um caminho de 10 Gb/s como equivalente a um de 100 Mb/s (por exemplo, poderia fazer ECMP entre eles ou preferir um caminho com menos saltos mas mais lento). Isso só é corrigido com `auto-cost reference-bandwidth <Mb/s>` no processo OSPF (por exemplo 1000 ou 10000), que deve ser igual em todos os roteadores do domínio, ou fixando o custo à mão com `ip ospf cost`. Sem reconfigurar, o OSPF não enxerga a diferença.

### Q6. O batimento cardíaco do OSPF

**Intervalos configurados.** Em todas as interfaces OSPF do triângulo (as saídas completas estão na Q4), a linha de timers é a mesma:

```
MTZ-R1>show ip ospf interface g0/1
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
```

O **Hello** é de **10 s** e o **Dead** de **40 s** (quatro vezes o Hello), os valores padrão para redes do tipo broadcast. As LANs (g0/0), por serem passivas, não enviam Hello (`No Hellos (Passive interface)`).

**Medição no modo Simulation.** Filtro só em OSPF. Evidência: `evidencias/prints/q6_hellos.png`, com a Event List e a coluna Time(sec). Dois Hellos consecutivos enviados pelo MTZ-R1 ao MTZ-R3 pelo ENL2 (IP de origem 10.1.2.157, destino multicast 224.0.0.5, MAC 0100.5E00.0005, mensagem *OSPF HELLO*; a aba OSI Model do MTZ-R3 diz que o Hello vem de um vizinho existente e que o dispositivo reinicia os temporizadores desse vizinho):

| Evento | Last Device → At Device | Time(sec) |
| :--- | :--- | :--- |
| Hello 1 | MTZ-R1 → MTZ-R3 | 0.635 |
| Hello 2 | MTZ-R1 → MTZ-R3 | 10.637 |
| **Diferença** | | **10,002 s ≈ 10 s** |

A diferença confirma o intervalo de Hello de 10 s (os 0,002 s a mais são o atraso de propagação da simulação).

**O que acontece quando um roteador fica um intervalo Dead inteiro sem ser ouvido.** Cada Hello recebido reinicia o temporizador de Dead do vizinho (é o contador visto na coluna *Dead Time* do `show ip ospf neighbor`, que decresce de 40 s e volta a 40 s a cada Hello). Se o contador chega a zero, ou seja, passam 40 s (quatro Hellos perdidos seguidos) sem nenhum Hello desse vizinho, o roteador o declara **inativo**: a adjacência é desfeita (o vizinho some da tabela de vizinhos), o roteador remove esse enlace da sua LSDB (gera um novo LSA sem ele), e todos os roteadores da área refazem o cálculo SPF e atualizam a tabela de rotas, passando a usar caminhos alternativos (por exemplo, ir a um destino pelo outro lado do triângulo). Se o vizinho perdido era o DR ou o BDR do enlace, uma nova eleição ocorre nesse segmento.

## Etapa 2

### Q7
Os quatro relógios do RIP. Cole o show ip protocols de FIL-R1 e extraia os valores dos ti‐
mers de update, invalid, holddown e flush. Explique em uma frase o papel de cada um e o
que significa a linha "next update due in...".
Onde procurar: CLI de FIL-R1, primeiro bloco da saída de show ip protocols.
---
```
FIL-R1#show ip protocols
Routing Protocol is "rip"
Sending updates every 30 seconds, next due in 27 seconds
Invalid after 180 seconds, hold down 180, flushed after 240
Outgoing update filter list for all interfaces is not set
Incoming update filter list for all interfaces is not set
Redistributing: rip
Default version control: send version 2, receive 2
Interface Send Recv Triggered RIP Key-chain
Serial0/0/0 22
Automatic network summarization is not in effect
Maximum path: 4
Routing for Networks:
10.0.0.0
Passive Interface(s):
GigabitEthernet0/0
Routing Information Sources:
Gateway Distance Last Update
10.1.2.178 120 00:00:17
Distance: (default is 120)
```
| Nome | tempo |
|--|--|
| update | 30 segundos
| invalid | 180 segundos
| holddown | 180 segundos
| flush | 240 segundos

O valor de "update" é a frequência com que o roteador vai mandar a sua tabela de distâncias. 

O valor de "invalid" referente ao tempo em que uma rota pode ficar sem receber updates e ainda ser considerada valida. Depois de "invalid" segundos sem receber nenhum update da rota ela se torna invalida.

O valor de "holddown" é um threshold para atualizações que pioram a tabela atual, de forma a evitar mudança excessiva na tabela.

O valor de "flush" é o limite para a rota ser removida da tabela (só é contado a partir do momento que a rota se torna invalida)

A frase "next due in 27 seconds" é a contagem de segundos que faltam para o próximo update ser enviado para os a rede.

### Q8
Por dentro de um Response. No modo Simulation, capture um RIP Response periódico de FIL-R1. Para qual endereço de destino ele é enviado, e por que RIPv2 usa multicast em vez do broadcast do RIPv1? Liste os campos presentes em cada entrada de rota do Response e diga qual deles é a razão de o RIPv2 suportar VLSM.
Onde procurar: modo Simulation com filtro só em RIP. Outbound PDU Details do Response (destino 224.0.0.9, procure o campo de máscara nas entradas). Se os PDU Details da sua versão não exibirem o campo de máscara nas entradas de rota, use como evidência o show ip route de FIL-R2, que mostra subredes de tamanhos diferentes convivendo dentro da mesma rede 10.0.0.0, e explique por que isso seria impossível com RIPv1.
---

O endereço de destino é o 224.0.0.9 (multicast), o multicast é utilizado para evitar enviar os pacotes RIPv2 para dispositivos que não estão participando do protocolo, diferente do RIPv1 que usa o broadcast padrão 255.255.255.255.

Os campos presentes são: (seguindo o q8_rip_response.png)
- ADDRESS FAMILY: 2
- ROUTE TAG: 0
- NETWORK ADDRESS: 10.1.2.96
- SUBNET MASK: 255.255.255.224
- NEXT HOP: 10.1.2.177
- METRIC: 1

O campo que permite a utilização de VLSM é o SUBNET MASK em conjunto com o NETWORK ADDRESS.
Como o RIPv1 não envia a mascara da subrede, o roteador que recebe o rip response precisa supor uma mascara para colocar em sua tabela de roteamento, podendo causar conflitos e inconsistencia.


### Q9
Métrica em saltos. Na tabela de FIL-R2, interprete a rota R para a LAN-FIL1: o que significa [120/1]? Por que a métrica do RIP é contada em saltos, e qual a consequência prática do limite de 15?
Onde procurar: show ip route rip em FIL-R2.
--- 

```
Gateway of last resort is not set

10.0.0.0/8 is variably subnetted, 5 subnets, 4 masks
R 10.1.2.96/27 [120/1] via 10.1.2.177, 00:00:09, Serial0/0/0
C 10.1.2.128/28 is directly connected, GigabitEthernet0/0
L 10.1.2.129/32 is directly connected, GigabitEthernet0/0
C 10.1.2.176/30 is directly connected, Serial0/0/0
L 10.1.2.178/32 is directly connected, Serial0/0/0
```

A primeira mensagem é referente a não ter sido configurado uma saída padrão para o roteador, caso o ip que queremos nos comunicar não esteja presente em alguma das redes alcançaveis, a saída padrão seria usada.
A Rota R para 10.1.2.96/27 é uma rota RIP, as rotas L e C são local e connected respectivamente.

A métrica 120/1 é no formato [distância administrativa/hop], a distância administrativa padrão é 120, e o número de roteadores entre o FIL-R2 e a rede 10.1.2.96/27 é 1.

A consequência prática de um limite de hops em 15 é que em casos de uma rota cair e ser atualizada de forma incorreta pela falta de conhecimento topolgócio sobre a rede nos roteadores, a rota será removida da tabela quando atingir a métrica 16.

### Q10
Tagarelice comparada. Com as duas redes estáveis (sem qualquer mudança), compare o que RIP e OSPF continuam transmitindo periodicamente: o quê, com que frequência e de que tamanho? Use os tamanhos reais dos pacotes observados para estimar quantos bytes por minuto cada protocolo gasta em um enlace parado, e conclua qual dos dois cresce quando a rede ganha novas sub-redes. Atenção: com o número pequeno de rotas desta topologia os dois valores medidos ficam próximos, e podem até inverter. A conclusão pedida é sobre o comportamento de cada protocolo à medida que a tabela de rotas cresce, não sobre qual número é maior nesta medição.
Onde procurar: modo Simulation, uma rodada com filtro OSPF (na matriz) e outra com filtro RIP (nafilial). O tamanho do pacote aparece nos PDU Details e a periodicidade, na coluna Time.
---

A rede RIP esta configurada com update=30, logo cada roteador envia 2 rip responses por minuto (como pode ser visto em q10_rip_periodico), onde cada um possui tamanho 52 bytes

A rede OSPF esta configurada para um HELLO a cada 10 segundos, logo cada enlace recebera 12 pacotes HELLO que possuem o tamanho de 20 bytes. 

Neste exemplo notamos que a rede RIP possui menos trafego, porém a medida que a quantidade de sub redes aumenta, os pacotes se tornam maiores e a frequencia de 30 supera o trafego normal da rede OSPF. 

Isso se da ao fato da rede OSPF ter uma frequencia menor para atualização topológica.

## Etapa 3

### Q11. O network classful do RIP

O RIP trabalha com o conceito de classes de endereços, não com prefixos exatos. O comando `network 10.0.0.0` não significa "ative o RIP somente nesta sub-rede /8": significa "ative o RIP em todas as interfaces cujo endereço IP pertence à classe A 10.0.0.0/8". No R-BORDA, todas as interfaces com IP têm endereços dentro de 10.0.0.0/8 (ENL4 = 10.1.2.166/30, ENL5 = 10.1.2.170/30 e ENL6 = 10.1.2.173/30), portanto o `network 10.0.0.0` as ativa todas para o RIP.

O problema é que as interfaces ENL4 e ENL5 (seriais s0/3/0 e s0/3/1) conectam o R-BORDA à matriz OSPF. Se o RIP ficasse ativo nessas interfaces, o R-BORDA tentaria enviar e receber RIP Updates nesses enlaces — trocando tabelas de roteamento com MTZ-R2 e MTZ-R3, que não executam RIP e ignorariam os pacotes. Além disso, roteadores que falam apenas OSPF na matriz não esperariam receber mensagens RIP, o que seria tráfego desnecessário e potencialmente confuso. O comportamento correto é que ENL4 e ENL5 sirvam somente para o OSPF (configurado explicitamente com `network 10.1.2.164 0.0.0.3 area 0` e `network 10.1.2.168 0.0.0.3 area 0`), enquanto ENL6 (serial s0/2/0, que liga ao FIL-R1) é o único enlace onde o RIP deve de fato enviar e receber Updates.

O `passive-interface` resolve exatamente esse problema. Quando uma interface é declarada passiva no RIP, o roteador para de enviar RIP Updates por ela — ele continua incluindo as redes dessa interface em suas mensagens e recebendo Updates chegando por ela, mas não envia Updates ativos. No R-BORDA, `passive-interface Serial0/3/0` e `passive-interface Serial0/3/1` silenciam o RIP nas duas seriais que conectam à matriz. Assim ENL4 e ENL5 deixam de receber tráfego RIP do R-BORDA, e apenas a ENL6 (s0/2/0, que leva ao FIL-R1) continua enviando Updates RIP — que é o comportamento desejado.

Do lado do OSPF, a lógica é inversa: a s0/2/0 é declarada `passive-interface` no processo OSPF para que o R-BORDA não tente formar adjacência OSPF com o FIL-R1 (que não fala OSPF). O R-BORDA é, portanto, um Autonomous System Boundary Router (ASBR): fala OSPF com a matriz pelas seriais ENL4 e ENL5, e RIP com a filial pela serial ENL6, com cada protocolo bloqueado nas interfaces do outro domínio via `passive-interface`.

---

### Q12. A métrica-semente que faltou

Quando dois protocolos de roteamento coexistem e um roteador precisa anunciar rotas de um domínio para o outro, é preciso atribuir uma métrica inicial às rotas redistribuídas — a chamada **métrica-semente**. Sem ela, o protocolo receptor não sabe qual custo atribuir às rotas importadas e as descarta ou as trata como inacessíveis.

No RIP, a inacessibilidade é representada pela métrica **16** (infinito). Na primeira tabela exibida, o FIL-R1 não tem as rotas da matriz: as únicas entradas são as redes diretamente conectadas e a LAN-FIL2 (10.1.2.128/28) aprendida do FIL-R2 via RIP com métrica 1. Isso significa que, naquele momento, a redistribuição `redistribute ospf 1` estava configurada no R-BORDA **sem** a palavra-chave `metric 3`. O RIPv2 usa métrica padrão infinita (16) para rotas redistribuídas sem métrica explícita, e rotas com métrica 16 são descartadas como inalcançáveis pelos vizinhos — por isso o FIL-R1 simplesmente não as aprende.

Depois de adicionar `metric 3` ao comando (`redistribute ospf 1 metric 3`), o R-BORDA passa a injetar cada rota da matriz no RIP com métrica inicial 3. O valor 3 foi escolhido para representar, de forma aproximada, a distância do R-BORDA até as redes internas da matriz: o R-BORDA já está a 1 ou 2 saltos das LANs e enlances da matriz via OSPF, e a métrica-semente de 3 resulta em métricas finais coerentes para o FIL-R1 (3 saltos até redes como LAN-ENG ou LAN-ADM, já que estão "atrás" do R-BORDA) e para o FIL-R2 (4 saltos, que é FIL-R1 + 1).

Na segunda tabela, o FIL-R1 passa a ver todas as redes da matriz com métrica [120/3] via 10.1.2.173 (R-BORDA, pelo ENL6), e o FIL-R2 (que aparece na Q14) as vê com [120/4] via 10.1.2.177 (FIL-R1, pelo ENL7). As redes ENL4 (10.1.2.164/30) e ENL5 (10.1.2.168/30) aparecem com [120/1], porque são as próprias seriais do R-BORDA que fazem fronteira — estão a 1 salto do FIL-R1.

---

### Q13. Rotas de segunda mão no OSPF

O OSPF distingue dois tipos de rotas externas ao domínio: **E1** e **E2**. No contexto deste trabalho, as rotas das LANs da filial (LAN-FIL1 = 10.1.2.96/27, LAN-FIL2 = 10.1.2.128/28) e das seriais da filial (ENL6 = 10.1.2.172/30, ENL7 = 10.1.2.176/30) foram redistribuídas do RIP para o OSPF no R-BORDA via `redistribute rip subnets`. Por padrão, rotas redistribuídas no OSPF recebem o tipo **E2**, e é por isso que a tabela de MTZ-R1 e MTZ-R3 exibe o código `O E2` para essas redes.

**O que significa `O E2`.** O `O` indica que a rota foi aprendida por OSPF. O `E2` (External Type 2) significa que a métrica exibida é a **métrica-semente fixada no ponto de redistribuição** — no caso, 20, que é o valor padrão quando o OSPF redistribui rotas externas sem `metric` explícito — e essa métrica **não muda** à medida que o pacote atravessa roteadores OSPF internos. Independentemente de o MTZ-R1 estar a 1 ou a 10 saltos do R-BORDA, a métrica reportada continua 20. O tipo E1 somaria ao custo externo o custo interno acumulado dentro do OSPF, o que não é o caso aqui.

**A notação [110/20].** Os dois números têm o mesmo significado de sempre: `110` é a distância administrativa do OSPF, e `20` é a métrica OSPF da rota. Para rotas E2, essa métrica é a métrica-semente da redistribuição. O valor 20 é o padrão adotado pelo Cisco IOS para redistribuição no OSPF quando nenhuma métrica é especificada no comando `redistribute`.

**Por que as rotas internas da filial aparecem em todos os roteadores OSPF da matriz.** O R-BORDA é um ASBR (Autonomous System Boundary Router): ao redistribuir do RIP para o OSPF, ele gera LSAs do tipo 5 (AS External LSA), que são propagados por toda a área 0 sem restrição. Assim, MTZ-R1, MTZ-R2 e MTZ-R3 todos aprendem as redes da filial como rotas E2, com métrica 20, e alcançam o R-BORDA pelo caminho de menor custo OSPF disponível — daí o ECMP visto em MTZ-R1 (dois next-hops iguais para as redes externas: 10.1.2.154 via g0/1 e 10.1.2.158 via g0/2, ambas chegando ao R-BORDA por caminhos de mesmo custo).

**As rotas internas da matriz no MTZ-R3 (`O` sem `E2`).** As rotas marcadas apenas com `O` (como LAN-ENG 10.1.2.64/27 e ENL1 10.1.2.152/30) são rotas OSPF internas, aprendidas por LSAs tipo 1 e 2 dentro da própria área 0 — sem redistribuição. Essas têm custo real calculado pelo SPF e são mais confiáveis para o roteador do que as externas E2.

---

### Q14. Rotas de segunda mão no RIP

A Q14 mostra o outro lado da redistribuição: as rotas da matriz OSPF vistas pelos roteadores da filial via RIP.

**No FIL-R1.** Todas as rotas da matriz aparecem com distância administrativa 120 (padrão do RIP) e métricas RIP que partem de 3. A métrica 3 vem da métrica-semente configurada no R-BORDA (`redistribute ospf 1 metric 3`). As redes do interior da matriz — LAN-ADM (10.1.2.0/26), LAN-ENG (10.1.2.64/27), LAN-SRV (10.1.2.144/29) e os enlaces do triângulo ENL1, ENL2, ENL3 (10.1.2.152–10.1.2.160/30) — chegam com [120/3], indicando 3 saltos RIP a partir do R-BORDA. Os enlaces ENL4 e ENL5 (10.1.2.164/30 e 10.1.2.168/30), que são as próprias interfaces seriais do R-BORDA que conectam à matriz, chegam com [120/1] — porque essas redes estão diretamente conectadas ao R-BORDA, que as anuncia ao FIL-R1 com apenas 1 salto adicionado à métrica inicial de 0 (elas não são redistribuídas do OSPF, são diretamente conectadas ao próprio R-BORDA e aprendidas nativamente pelo RIP via `network 10.0.0.0`).

**No FIL-R2.** O FIL-R2 aprende tudo a partir do FIL-R1: cada rota ganha +1 em relação ao que o FIL-R1 enxerga. As redes internas da matriz que chegavam ao FIL-R1 com métrica 3 chegam ao FIL-R2 com métrica 4 ([120/4] via 10.1.2.177, que é o FIL-R1 no ENL7). As redes ENL4 e ENL5, que chegavam ao FIL-R1 com métrica 1, chegam ao FIL-R2 com métrica 2. A LAN-FIL1 (10.1.2.96/27), que é diretamente conectada ao FIL-R1, chega ao FIL-R2 com [120/1]. O ENL6 (10.1.2.172/30, entre R-BORDA e FIL-R1) aparece no FIL-R2 com [120/1] porque o FIL-R1 o anuncia como rede diretamente conectada; já no FIL-R1 essa rede é `C` (connected) e não aparece como rota RIP.

**Assimetria importante.** O FIL-R1 não tem rota para a LAN-FIL2 (10.1.2.128/28) com via 10.1.2.173 (R-BORDA) — essa rede chega a ele via FIL-R2 ([120/1] via 10.1.2.178). Da mesma forma, o FIL-R2 não tem a LAN-FIL1 (10.1.2.96/27) via R-BORDA, mas sim via FIL-R1. Isso mostra que o RIP propagou corretamente as rotas das LANs das filiais entre si pelos ENL7.

---

### Q15. A jornada completa de um DISCOVER

Com a integração da Etapa 3, o PC-FIL2-1 (LAN-FIL2, 10.1.2.128/28) está em DHCP e o servidor centralizado SRV-DC (10.1.2.146) está na LAN-SRV (10.1.2.144/29) na matriz. O DISCOVER precisa atravessar dois domínios de roteamento distintos — RIP na filial e OSPF na matriz — para chegar ao servidor. Evidências: `evidencias/prints/q15_discover_jornada.png` e `evidencias/prints/q15_offer_retorno.png`.

**Dispositivos que o DISCOVER atravessa (em ordem):**

1. **PC-FIL2-1** emite o DISCOVER como broadcast (0.0.0.0 → 255.255.255.255, MAC FFFF.FFFF.FFFF) na LAN-FIL2.
2. **FIL-R2** recebe o broadcast na g0/0 (10.1.2.129). Por ter `ip helper-address 10.1.2.146` nessa interface, captura o pacote DHCP, preenche o campo GIADDR com 10.1.2.129 (IP da g0/0 na LAN-FIL2) e o reenvia como unicast em direção ao SRV-DC (10.1.2.146). O próximo salto para 10.1.2.146 na tabela do FIL-R2 é 10.1.2.177 (FIL-R1, via Serial0/0/0, ENL7) — rota aprendida via RIP.
3. **FIL-R1** recebe o pacote unicast na Serial0/0/0. Consulta sua tabela de rotas: 10.1.2.146 pertence à rede 10.1.2.144/29, que o FIL-R1 conhece via RIP com próximo salto 10.1.2.173 (R-BORDA, via Serial0/0/1, ENL6). Encaminha o pacote.
4. **R-BORDA** recebe o pacote na Serial0/2/0. Consulta sua tabela de rotas: 10.1.2.144/29 é uma rota OSPF (aprendida via os enlaces ENL4 ou ENL5), com próximo salto para a matriz. O R-BORDA encaminha pelo ENL4 ou ENL5 (seriais para MTZ-R2 ou MTZ-R3 respectivamente, que possuem custo igual — ECMP possível).
5. **MTZ-R2** (ou MTZ-R3) recebe o pacote e o encaminha para **MTZ-R1**, que tem a LAN-SRV como rede diretamente conectada (g0/0).
6. **MTZ-R1** entrega o pacote ao **SRV-DC** (10.1.2.146) na LAN-SRV.

**Caminho de volta do OFFER:**

O SRV-DC envia o OFFER em unicast para o GIADDR (10.1.2.129 — o FIL-R2). O caminho é o inverso: SRV-DC → MTZ-R1 → (MTZ-R2 ou MTZ-R3) → R-BORDA → FIL-R1 → FIL-R2. O FIL-R2, ao receber o OFFER destinado ao GIADDR 10.1.2.129 (sua própria interface), o retransmite em broadcast para a LAN-FIL2, onde o PC-FIL2-1 aguarda.

**Protocolo de roteamento usado em cada trecho:**

- **FIL-R2 → FIL-R1 (ENL7):** RIP. O FIL-R2 aprendeu a rota para 10.1.2.144/29 com métrica [120/4] via FIL-R1 (rota redistribuída da matriz via RIP).
- **FIL-R1 → R-BORDA (ENL6):** RIP. O FIL-R1 aprendeu 10.1.2.144/29 com [120/3] via R-BORDA (10.1.2.173).
- **R-BORDA → MTZ-R2 ou MTZ-R3 (ENL4 ou ENL5):** OSPF. O R-BORDA aprendeu 10.1.2.144/29 como rota OSPF interna (O) com custo baixo.
- **MTZ-R2/R3 → MTZ-R1 (ENL1 ou ENL2):** OSPF. A LAN-SRV é anunciada diretamente pelo MTZ-R1 no OSPF.

---

### Q16. Falha com rede viva

**ECMP no R-BORDA antes do shutdown.** O R-BORDA possui duas seriais para a matriz: ENL4 (s0/3/0 → MTZ-R2) e ENL5 (s0/3/1 → MTZ-R3). Ambas as rotas para redes internas da matriz, como o ENL3 (10.1.2.160/30), aparecem com o mesmo custo OSPF ([110/65]) — o custo 65 reflete o custo serial (referência 100 Mbps / 1,544 Mbps ≈ 64) somado ao custo de 1 do enlace GigE de destino. Como os dois caminhos têm custo idêntico, o IOS instala ambos na tabela de rotas (ECMP) e distribui o tráfego entre eles.

**Teste de falha da serial ENL4 (s0/3/0 do MTZ-R2).** Antes do shutdown, o `tracert` do PC-FIL2-1 para o SRV-DC (10.1.2.146) usa o caminho: FIL-R2 (10.1.2.129) → FIL-R1 (10.1.2.177) → R-BORDA (10.1.2.173) → **MTZ-R2 (10.1.2.165)**, salto 4 — via ENL4 — → MTZ-R1 (10.1.2.153) → SRV-DC (10.1.2.146), 6 saltos total.

Após o `shutdown` na serial ENL4 (s0/3/0 do MTZ-R2), o OSPF detecta a queda: o temporizador Dead (40 s padrão em seriais) expira sem Hellos do vizinho, a adjacência cai e um novo LSA é gerado removendo esse enlace. O R-BORDA recalcula o SPF e instala apenas o caminho pelo ENL5 (s0/3/1 → MTZ-R3). O novo `tracert` confirma: o salto 4 muda de 10.1.2.165 (MTZ-R2 no ENL4) para **10.1.2.169 (MTZ-R3 no ENL5)**, e o próximo salto dentro da matriz no salto 5 passa de 10.1.2.153 (MTZ-R1 via g0/1) para 10.1.2.157 (MTZ-R1 via g0/2) — o tráfego chega ao mesmo destino final pelo caminho alternativo, sem nenhuma intervenção manual.

**O DHCP funciona com a serial derrubada.** O `ipconfig /renew` no PC-FIL2-1 retorna endereço 10.1.2.132, máscara 255.255.255.240, gateway 10.1.2.129 — tudo correto para a LAN-FIL2 (10.1.2.128/28). Isso prova que o DISCOVER chegou ao SRV-DC pelo caminho alternativo (via ENL5), o servidor respondeu e o relay no FIL-R2 entregou o OFFER de volta ao PC. A resiliência da rede a uma falha de enlace funciona de ponta a ponta, inclusive para serviços que dependem de roteamento correto na ida e na volta (como DHCP com relay).

---

### Q17. A fronteira não é simétrica

A Q17 pede a análise comparada das tabelas de rotas do MTZ-R1 (dentro do domínio OSPF) e do FIL-R2 (dentro do domínio RIP), observando como cada um vê as redes do domínio alheio — e por que não existe simetria perfeita entre os dois lados da fronteira.

**Como o MTZ-R1 vê a filial.** O MTZ-R1 enxerga as redes da filial (LAN-FIL1 10.1.2.96/27, LAN-FIL2 10.1.2.128/28, ENL6 10.1.2.172/30 e ENL7 10.1.2.176/30) como rotas `O E2` com métrica [110/20]. Essas rotas chegam via redistribuição: o R-BORDA importou as redes do RIP para o OSPF com `redistribute rip subnets`, gerando LSAs tipo 5 com métrica padrão 20. O MTZ-R1 não sabe nada sobre as métricas RIP internas dessas redes — só sabe que elas existem e que o caminho passa pelo R-BORDA. Em ECMP, o MTZ-R1 usa dois next-hops (10.1.2.154 via g0/1 rumo ao MTZ-R2, e 10.1.2.158 via g0/2 rumo ao MTZ-R3) porque ambos os caminhos até o R-BORDA têm custo OSPF igual. As redes ENL4 (10.1.2.164/30) e ENL5 (10.1.2.168/30) aparecem como rotas `O` comuns (não E2) com custo 65, porque são interfaces diretamente conectadas do R-BORDA anunciadas pelo próprio R-BORDA no OSPF.

**Como o FIL-R2 vê a matriz.** O FIL-R2 enxerga as redes da matriz como rotas `R` (RIP) com métrica [120/4] via 10.1.2.177 (FIL-R1). Ele não distingue se a origem foi OSPF ou RIP — para ele, são rotas RIP comuns, redistribuídas pelo R-BORDA com métrica-semente 3 e acrescidas de +1 por cada salto RIP até chegar ao FIL-R2. A única informação disponível é a métrica em saltos, não o custo real nem a topologia interna do OSPF.

**Por que a redistribuição não é simétrica.** No sentido OSPF → RIP, o R-BORDA injeta as rotas com `redistribute ospf 1 metric 3`: cada rede OSPF entra no RIP com métrica 3, independentemente do custo OSPF original. No sentido RIP → OSPF, o R-BORDA injeta com `redistribute rip subnets`: cada rede RIP entra no OSPF como rota E2 com métrica padrão 20. As métricas dos dois protocolos são incompatíveis entre si (custo de banda no OSPF vs. contagem de saltos no RIP), então o ponto de redistribuição precisa fixar um valor arbitrário em cada direção — e a escolha dessas métricas-semente (3 para OSPF→RIP, 20 para RIP→OSPF) é uma decisão de projeto.

**A assimetria no `network 10.0.0.0` e o `passive-interface`.** O FIL-R2 não tem `passive-interface` nas seriais — ele envia e recebe RIP em todas as interfaces ativas. O MTZ-R1, por outro lado, nunca viu o RIP: ele só fala OSPF. O R-BORDA é quem faz a tradução entre os dois mundos, operando como ASBR e usando `passive-interface` para garantir que cada protocolo só cruze as interfaces do seu próprio domínio (conforme explicado na Q11).

---
