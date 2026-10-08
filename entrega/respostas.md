# Respostas: Trabalho do Grau A (Redes)

**Integrantes:** [nome completo e matrícula de cada integrante]

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
