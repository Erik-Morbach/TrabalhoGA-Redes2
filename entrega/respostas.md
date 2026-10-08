# Respostas: Trabalho do Grau A (Redes)

**Integrantes:** Erik Cruz Morbach(1946271), Lucas Hoffmeister Escopelli(), Vinicius Muller Silveira(1789484)

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

Em R-BORDA, o comando `network 10.0.0.0` sob `router rip` ativa o RIP em quais interfaces? Por que isso é um problema em um roteador de borda cujas interfaces estão todas dentro de 10.0.0.0/8, e o que exatamente o `passive-interface` contém?

**Resposta:**

```
R-BORDA>show ip protocols
Routing Protocol is "rip"
Sending updates every 30 seconds, next due in 5 seconds
Invalid after 180 seconds, hold down 180, flushed after 240
Outgoing update filter list for all interfaces is not set
Incoming update filter list for all interfaces is not set
Redistributing: rip, ospf 1 
Default version control: send version 2, receive 2
  Interface             Send  Recv  Triggered RIP  Key-chain
  Serial0/2/0           22
Automatic network summarization is not in effect
Maximum path: 4
Routing for Networks:
	10.0.0.0
Passive Interface(s):
	Serial0/3/0
	Serial0/3/1
Routing Information Sources:
	Gateway         Distance      Last Update
	10.1.2.174           120      00:00:18
Distance: (default is 120)

Routing Protocol is "ospf 1"
  Outgoing update filter list for all interfaces is not set 
  Incoming update filter list for all interfaces is not set 
  Router ID 1.1.1.4
  It is an autonomous system boundary router
  Redistributing External Routes from,
    rip 
  Number of areas in this router is 1. 1 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
    10.1.2.164 0.0.0.3 area 0
    10.1.2.168 0.0.0.3 area 0
  Passive Interface(s): 
    Serial0/2/0
  Routing Information Sources:  
    Gateway         Distance      Last Update 
    1.1.1.1              110      00:13:50
    1.1.1.2              110      00:22:04
    1.1.1.3              110      00:22:03
    1.1.1.4              110      00:25:07
  Distance: (default is 110)

```

---

### Q12. A métrica-semente que faltou

**Sintoma (metric 16):**

```
FIL-R1>show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 5 subnets, 4 masks
C       10.1.2.96/27 is directly connected, GigabitEthernet0/0
L       10.1.2.97/32 is directly connected, GigabitEthernet0/0
R       10.1.2.128/28 [120/1] via 10.1.2.178, 00:00:14, Serial0/0/0
C       10.1.2.176/30 is directly connected, Serial0/0/0
L       10.1.2.177/32 is directly connected, Serial0/0/0
```

**Resultado correto (metric 3):**

```
FIL-R1>show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 15 subnets, 6 masks
R       10.1.2.0/26 [120/3] via 10.1.2.173, 00:00:12, Serial0/0/1
R       10.1.2.64/27 [120/3] via 10.1.2.173, 00:00:12, Serial0/0/1
C       10.1.2.96/27 is directly connected, GigabitEthernet0/0
L       10.1.2.97/32 is directly connected, GigabitEthernet0/0
R       10.1.2.128/28 [120/1] via 10.1.2.178, 00:00:12, Serial0/0/0
R       10.1.2.144/29 [120/3] via 10.1.2.173, 00:00:12, Serial0/0/1
R       10.1.2.152/30 [120/3] via 10.1.2.173, 00:00:12, Serial0/0/1
R       10.1.2.156/30 [120/3] via 10.1.2.173, 00:00:12, Serial0/0/1
R       10.1.2.160/30 [120/3] via 10.1.2.173, 00:00:12, Serial0/0/1
R       10.1.2.164/30 [120/1] via 10.1.2.173, 00:00:12, Serial0/0/1
R       10.1.2.168/30 [120/1] via 10.1.2.173, 00:00:12, Serial0/0/1
C       10.1.2.172/30 is directly connected, Serial0/0/1
L       10.1.2.174/32 is directly connected, Serial0/0/1
C       10.1.2.176/30 is directly connected, Serial0/0/0
L       10.1.2.177/32 is directly connected, Serial0/0/0
```

**Explicação:**

---

### Q13. Rotas de segunda mão no OSPF

```
MTZ-R1>show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 15 subnets, 6 masks
O       10.1.2.0/26 [110/2] via 10.1.2.158, 02:46:21, GigabitEthernet0/2
O       10.1.2.64/27 [110/2] via 10.1.2.154, 02:46:31, GigabitEthernet0/1
O E2    10.1.2.96/27 [110/20] via 10.1.2.154, 00:38:46, GigabitEthernet0/1
                     [110/20] via 10.1.2.158, 00:38:46, GigabitEthernet0/2
O E2    10.1.2.128/28 [110/20] via 10.1.2.154, 00:38:46, GigabitEthernet0/1
                      [110/20] via 10.1.2.158, 00:38:46, GigabitEthernet0/2
C       10.1.2.144/29 is directly connected, GigabitEthernet0/0
L       10.1.2.145/32 is directly connected, GigabitEthernet0/0
C       10.1.2.152/30 is directly connected, GigabitEthernet0/1
L       10.1.2.153/32 is directly connected, GigabitEthernet0/1
C       10.1.2.156/30 is directly connected, GigabitEthernet0/2
L       10.1.2.157/32 is directly connected, GigabitEthernet0/2
O       10.1.2.160/30 [110/2] via 10.1.2.154, 02:46:21, GigabitEthernet0/1
                      [110/2] via 10.1.2.158, 02:46:21, GigabitEthernet0/2
O       10.1.2.164/30 [110/65] via 10.1.2.154, 01:28:45, GigabitEthernet0/1
O       10.1.2.168/30 [110/65] via 10.1.2.158, 01:27:32, GigabitEthernet0/2
O E2    10.1.2.172/30 [110/20] via 10.1.2.154, 00:57:19, GigabitEthernet0/1
                      [110/20] via 10.1.2.158, 00:57:19, GigabitEthernet0/2
O E2    10.1.2.176/30 [110/20] via 10.1.2.154, 00:38:46, GigabitEthernet0/1
                      [110/20] via 10.1.2.158, 00:38:46, GigabitEthernet0/2
```

```
MTZ-R3>show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 16 subnets, 6 masks
C       10.1.2.0/26 is directly connected, GigabitEthernet0/0
L       10.1.2.1/32 is directly connected, GigabitEthernet0/0
O       10.1.2.64/27 [110/2] via 10.1.2.161, 02:46:51, GigabitEthernet0/2
O E2    10.1.2.96/27 [110/20] via 10.1.2.170, 00:39:11, Serial0/3/0
O E2    10.1.2.128/28 [110/20] via 10.1.2.170, 00:39:11, Serial0/3/0
O       10.1.2.144/29 [110/2] via 10.1.2.157, 02:46:51, GigabitEthernet0/1
O       10.1.2.152/30 [110/2] via 10.1.2.157, 02:46:51, GigabitEthernet0/1
                      [110/2] via 10.1.2.161, 02:46:51, GigabitEthernet0/2
C       10.1.2.156/30 is directly connected, GigabitEthernet0/1
L       10.1.2.158/32 is directly connected, GigabitEthernet0/1
C       10.1.2.160/30 is directly connected, GigabitEthernet0/2
L       10.1.2.162/32 is directly connected, GigabitEthernet0/2
O       10.1.2.164/30 [110/65] via 10.1.2.161, 01:29:10, GigabitEthernet0/2
C       10.1.2.168/30 is directly connected, Serial0/3/0
L       10.1.2.169/32 is directly connected, Serial0/3/0
O E2    10.1.2.172/30 [110/20] via 10.1.2.170, 00:57:44, Serial0/3/0
O E2    10.1.2.176/30 [110/20] via 10.1.2.170, 00:39:11, Serial0/3/0
```

**Explicação do código O E2 e da notação [110/20]:**

---

### Q14. Rotas de segunda mão no RIP

```
FIL-R1>show ip route rip
     10.0.0.0/8 is variably subnetted, 15 subnets, 6 masks
R       10.1.2.0/26 [120/3] via 10.1.2.173, 00:00:01, Serial0/0/1
R       10.1.2.64/27 [120/3] via 10.1.2.173, 00:00:01, Serial0/0/1
R       10.1.2.128/28 [120/1] via 10.1.2.178, 00:00:04, Serial0/0/0
R       10.1.2.144/29 [120/3] via 10.1.2.173, 00:00:01, Serial0/0/1
R       10.1.2.152/30 [120/3] via 10.1.2.173, 00:00:01, Serial0/0/1
R       10.1.2.156/30 [120/3] via 10.1.2.173, 00:00:01, Serial0/0/1
R       10.1.2.160/30 [120/3] via 10.1.2.173, 00:00:01, Serial0/0/1
R       10.1.2.164/30 [120/1] via 10.1.2.173, 00:00:01, Serial0/0/1
R       10.1.2.168/30 [120/1] via 10.1.2.173, 00:00:01, Serial0/0/1
```

```
FIL-R2>show ip route rip
     10.0.0.0/8 is variably subnetted, 14 subnets, 6 masks
R       10.1.2.0/26 [120/4] via 10.1.2.177, 00:00:10, Serial0/0/0
R       10.1.2.64/27 [120/4] via 10.1.2.177, 00:00:10, Serial0/0/0
R       10.1.2.96/27 [120/1] via 10.1.2.177, 00:00:10, Serial0/0/0
R       10.1.2.144/29 [120/4] via 10.1.2.177, 00:00:10, Serial0/0/0
R       10.1.2.152/30 [120/4] via 10.1.2.177, 00:00:10, Serial0/0/0
R       10.1.2.156/30 [120/4] via 10.1.2.177, 00:00:10, Serial0/0/0
R       10.1.2.160/30 [120/4] via 10.1.2.177, 00:00:10, Serial0/0/0
R       10.1.2.164/30 [120/2] via 10.1.2.177, 00:00:10, Serial0/0/0
R       10.1.2.168/30 [120/2] via 10.1.2.177, 00:00:10, Serial0/0/0
R       10.1.2.172/30 [120/1] via 10.1.2.177, 00:00:10, Serial0/0/0
```

---

### Q15. A jornada completa de um DISCOVER

Captura: 

**Dispositivos que o DISCOVER atravessa (em ordem):**

**Caminho de volta do OFFER:**

**Explicação do protocolo de roteamento usado em cada trecho:**

---

### Q16. Falha com rede viva

**ECMP no R-BORDA:**

```
O       10.1.2.160/30 [110/65] via 10.1.2.165, 01:49:46, Serial0/3/0
                      [110/65] via 10.1.2.169, 01:49:46, Serial0/3/1
```

**tracert 1 (antes do shutdown):**

```
Tracing route to 10.1.2.146 over a maximum of 30 hops: 

  1   0 ms      0 ms      0 ms      10.1.2.129
  2   0 ms      13 ms     0 ms      10.1.2.177
  3   8 ms      1 ms      20 ms     10.1.2.173
  4   15 ms     20 ms     6 ms      10.1.2.165
  5   15 ms     18 ms     6 ms      10.1.2.153
  6   5 ms      12 ms     1 ms      10.1.2.146
```

**tracert 2 (depois do shutdown):**

```
C:\>tracert 10.1.2.146

Tracing route to 10.1.2.146 over a maximum of 30 hops: 

  1   0 ms      0 ms      0 ms      10.1.2.129
  2   1 ms      0 ms      0 ms      10.1.2.177
  3   1 ms      19 ms     3 ms      10.1.2.173
  4   2 ms      27 ms     8 ms      10.1.2.169
  5   16 ms     13 ms     16 ms     10.1.2.157
  6   12 ms     15 ms     13 ms     10.1.2.146

Trace complete.
```

**ipconfig /renew com serial derrubada:**

```
C:\>ipconfig /renew

   IP Address......................: 10.1.2.132
   Subnet Mask.....................: 255.255.255.240
   Default Gateway.................: 10.1.2.129
   DNS Server......................: 0.0.0.0

```

**Explicação:**

---

### Q17. A fronteira não é simétrica

```
MTZ-R1>show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 15 subnets, 6 masks
O       10.1.2.0/26 [110/2] via 10.1.2.158, 03:22:27, GigabitEthernet0/2
O       10.1.2.64/27 [110/2] via 10.1.2.154, 03:22:37, GigabitEthernet0/1
O E2    10.1.2.96/27 [110/20] via 10.1.2.154, 00:00:32, GigabitEthernet0/1
                     [110/20] via 10.1.2.158, 00:00:32, GigabitEthernet0/2
O E2    10.1.2.128/28 [110/20] via 10.1.2.154, 00:00:32, GigabitEthernet0/1
                      [110/20] via 10.1.2.158, 00:00:32, GigabitEthernet0/2
C       10.1.2.144/29 is directly connected, GigabitEthernet0/0
L       10.1.2.145/32 is directly connected, GigabitEthernet0/0
C       10.1.2.152/30 is directly connected, GigabitEthernet0/1
L       10.1.2.153/32 is directly connected, GigabitEthernet0/1
C       10.1.2.156/30 is directly connected, GigabitEthernet0/2
L       10.1.2.157/32 is directly connected, GigabitEthernet0/2
O       10.1.2.160/30 [110/2] via 10.1.2.154, 03:22:27, GigabitEthernet0/1
                      [110/2] via 10.1.2.158, 03:22:27, GigabitEthernet0/2
O       10.1.2.164/30 [110/65] via 10.1.2.154, 00:00:42, GigabitEthernet0/1
O       10.1.2.168/30 [110/65] via 10.1.2.158, 02:03:38, GigabitEthernet0/2
O E2    10.1.2.172/30 [110/20] via 10.1.2.154, 00:00:32, GigabitEthernet0/1
                      [110/20] via 10.1.2.158, 00:00:32, GigabitEthernet0/2
O E2    10.1.2.176/30 [110/20] via 10.1.2.154, 00:00:32, GigabitEthernet0/1
                      [110/20] via 10.1.2.158, 00:00:32, GigabitEthernet0/2
```

```
FIL-R2>show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 14 subnets, 6 masks
R       10.1.2.0/26 [120/4] via 10.1.2.177, 00:00:20, Serial0/0/0
R       10.1.2.64/27 [120/4] via 10.1.2.177, 00:00:20, Serial0/0/0
R       10.1.2.96/27 [120/1] via 10.1.2.177, 00:00:20, Serial0/0/0
C       10.1.2.128/28 is directly connected, GigabitEthernet0/0
L       10.1.2.129/32 is directly connected, GigabitEthernet0/0
R       10.1.2.144/29 [120/4] via 10.1.2.177, 00:00:20, Serial0/0/0
R       10.1.2.152/30 [120/4] via 10.1.2.177, 00:00:20, Serial0/0/0
R       10.1.2.156/30 [120/4] via 10.1.2.177, 00:00:20, Serial0/0/0
R       10.1.2.160/30 [120/4] via 10.1.2.177, 00:00:20, Serial0/0/0
R       10.1.2.164/30 [120/2] via 10.1.2.177, 00:00:20, Serial0/0/0
R       10.1.2.168/30 [120/2] via 10.1.2.177, 00:00:20, Serial0/0/0
R       10.1.2.172/30 [120/1] via 10.1.2.177, 00:00:20, Serial0/0/0
C       10.1.2.176/30 is directly connected, Serial0/0/0
L       10.1.2.178/32 is directly connected, Serial0/0/0
```

**Explicação (redistribuição, network 10.0.0.0, passive-interface):**
