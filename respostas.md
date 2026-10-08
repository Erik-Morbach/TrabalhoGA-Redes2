## Etapa 1

Q1. A conversa DHCP completa. Com um PC da LAN-ENG configurado para DHCP, capture
no modo Simulation a troca completa de mensagens até o PC receber endereço. Monte uma
tabela com as quatro mensagens observadas na LAN do cliente (nome
DISCOVER/OFFER/REQUEST/ACK, IP de origem, IP de destino e MAC de destino) e explique
por que as mensagens do cliente partem de 0.0.0.0. Atenção: cada mensagem aparece
duas vezes na Event List, na versão do lado do cliente e na versão encaminhada entre MTZR2 e o servidor. Nesta questão use apenas a versão observada na LAN do cliente. A compa‐
ração entre as duas é o objeto de Q2.
Onde procurar: modo Simulation com filtro só em DHCP. Dispare a renovação com ipconfig /renew no
Command Prompt do PC. Clique em cada envelope e leia os campos em Inbound/Outbound PDU Details
(camadas 2 e 3 + seção DHCP/BOOTP).
Q2. O relay em ação. Compare o DHCPDISCOVER antes e depois de passar por MTZ-R2: o
que muda nos IPs de origem e destino? Qual campo do cabeçalho DHCP passa a carregar
um endereço da sub-rede do cliente, e como o servidor usa esse campo para escolher o
pool correto?
Onde procurar: no modo Simulation, clique no evento do pacote em MTZ-R2 e compare Inbound PDU De‐
tails com Outbound PDU Details. Procure o campo GIADDR (Relay Agent IP) na seção DHCP.
Q3. E sem o relay? Remova temporariamente o ip helper-address da interface LAN de MTZR2 (no ip helper-address ...), peça ipconfig /renew no PC e observe no modo Simulation
onde o DISCOVER morre. Explique por que um broadcast não atravessa um roteador e repo‐
nha a configuração.
Onde procurar: Event List: o pacote aparece chegando a MTZ-R2 e não sai dele. A aba OSI Model da PDU
Information em MTZ-R2 descreve o descarte.
Q4. DR e BDR onde não se esperava. Cole o show ip ospf neighbor de MTZ-R1 e explique
cada coluna. Por que um enlace GigabitEthernet com apenas dois roteadores ainda elege
DR e BDR? Compare o DR eleito em cada enlace do triângulo com os Router ID que o grupo
configurou: o roteador de maior Router ID venceu em todos os enlaces? Se não venceu em
algum, explique por qual mecanismo o OSPF manteve o DR já eleito e o que seria necessário
fazer para forçar uma nova eleição.
Onde procurar: CLI de MTZ-R1. O comando show ip ospf interface g0/1 mostra o Network Type
(Broadcast), a prioridade da interface e quem é DR e BDR. Registre também a ordem em que o grupo
configurou o OSPF em cada roteador, porque ela faz parte da resposta.

Q5. Anatomia de [110/2]. Na tabela de MTZ-R2, localize a rota para a LAN-SRV e decompo‐
nha a notação [110/2]: o que é cada número? Mostre a conta do custo somando os custos
das interfaces de saída no caminho e explique por que, com a referência de banda padrão
de 100 Mb/s, uma FastEthernet hipotética no lugar de qualquer enlace do triângulo teria exa‐
tamente o mesmo custo que a GigabitEthernet, e o que isso revela sobre a capacidade do
OSPF de distinguir enlaces acima de 100 Mb/s sem reconfiguração.
Onde procurar: show ip route em MTZ-R2 + show ip ospf interface <if> (linha Cost) nos roteadores
do caminho.
Q6. O batimento cardíaco do OSPF. Quais são os intervalos de Hello e Dead nas interfaces
OSPF? Confirme o intervalo de Hello medindo-o no modo Simulation (diferença entre os
tempos de dois Hellos consecutivos na Event List) e explique o que acontece quando um ro‐
teador fica um intervalo Dead inteiro sem ser ouvido.
Onde procurar: show ip ospf interface <if> (linha Timer intervals). Modo Simulation com filtro só em
OSPF, coluna Time.

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
