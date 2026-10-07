# Ordem de configuração do OSPF na matriz: MTZ-R1 → MTZ-R2 → MTZ-R3

A Q4 pede a ordem em que o grupo configurou o OSPF em cada roteador, porque ela faz parte da explicação da eleição de DR/BDR. Este arquivo registra a ordem usada e o que aconteceu em cada passo.

## Ordem usada
1. MTZ-R1 (router-id 1.1.1.1)
2. MTZ-R2 (router-id 1.1.1.2)
3. MTZ-R3 (router-id 1.1.1.3)

Em todos: `router-id` definido antes do primeiro `network`, `network` com wildcard exato, área 0, e `passive-interface g0/0` na LAN do roteador.

## O que aconteceu em cada passo

| Passo | Roteador configurado | Efeito observado |
| :--- | :--- | :--- |
| 1 | MTZ-R1 | Processo ativo, mas sem vizinhos. `show ip ospf neighbor` vazio, porque nenhum outro roteador respondia aos Hellos. |
| 2 | MTZ-R2 | Adjacência R1–R2 formada pelo ENL1. Saída vista no R2: vizinho 1.1.1.1 em `FULL/DR` na Gi0/1. |
| 3 | MTZ-R3 | Adjacência com R1 (ENL2) e com R2 (ENL3). O triângulo fica completo. |

## Por que a ordem importa para DR/BDR
- Em enlaces Ethernet o OSPF trata o segmento como multiacesso e elege DR e BDR, mesmo com apenas dois roteadores.
- A regra normal: vence a maior prioridade (padrão 1) e, em empate, o **maior router-id**.
- A eleição **não é preemptiva**. Quem já foi eleito continua DR mesmo se chegar depois um roteador com ID maior.
- No ENL1, o MTZ-R1 subiu primeiro, ficou sozinho no segmento e se elegeu DR. Quando o MTZ-R2 (ID maior) apareceu, o DR já existia e foi mantido. Por isso o R1, com o **menor** ID, é DR.
- O mesmo vale no ENL2: o MTZ-R1 já estava ativo quando o MTZ-R3 entrou e continuou DR (ver resultado abaixo).
- No ENL3 (R2–R3) o resultado depende de quem estava ativo quando o enlace subiu. Ainda precisa ser conferido com `show ip ospf interface g0/2` no MTZ-R2 ou no MTZ-R3.

## Resultado observado (MTZ-R1 → MTZ-R2 → MTZ-R3 configurados)

### Vizinhos do MTZ-R1
```
Router#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           1   FULL/BDR        00:00:35    10.1.2.154      GigabitEthernet0/1
1.1.1.3           1   FULL/BDR        00:00:32    10.1.2.158      GigabitEthernet0/2
```
O estado `FULL/BDR` descreve o papel do **vizinho** na rede. Os dois vizinhos são BDR, então o MTZ-R1 é o DR nos dois enlaces (ENL1 e ENL2), mesmo tendo o **menor** router-id.

### Interface g0/1 do MTZ-R1 (ENL1)
```
Router#show ip ospf interface g0/1

GigabitEthernet0/1 is up, line protocol is up
  Internet address is 10.1.2.153/30, Area 0
  Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 1.1.1.1, Interface address 10.1.2.153
  Backup Designated Router (ID) 1.1.1.2, Interface address 10.1.2.154
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
  ...
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 1.1.1.2  (Backup Designated Router)
```
- `Network Type BROADCAST`: mesmo com apenas dois roteadores, o enlace GigE faz eleição.
- `State DR`, `Priority 1`: o MTZ-R1 é o DR. A prioridade é igual à dos vizinhos, então o desempate seria pelo maior router-id, que não foi o que decidiu aqui.
- `Cost: 1` (GigabitEthernet) e `Hello 10, Dead 40`: valores usados na Q5 e na Q6.

### Resumo da eleição por enlace

| Enlace | Roteadores | DR | BDR | Venceu o maior router-id? |
| :--- | :--- | :--- | :--- | :--- |
| ENL1 (10.1.2.152/30) | R1 e R2 | MTZ-R1 (1.1.1.1) | MTZ-R2 (1.1.1.2) | Não |
| ENL2 (10.1.2.156/30) | R1 e R3 | MTZ-R1 (1.1.1.1) | MTZ-R3 (1.1.1.3) | Não (deduzido do `show ip ospf neighbor`) |
| ENL3 (10.1.2.160/30) | R2 e R3 | a conferir | a conferir | a conferir |

### `show ip route` no MTZ-R2: antes e depois do OSPF

**Antes:** só rotas conectadas (`C`) e locais (`L`), 6 sub-redes em 3 máscaras. O MTZ-R2 não conhece a LAN-SRV, a LAN-ADM nem o ENL2.

![show ip route no MTZ-R2 antes do OSPF](pics/mtz-r2-show-ip-route_antes%20ospf.png)

**Depois:** 9 sub-redes em 5 máscaras. Aparecem três rotas `O` novas (10.1.2.0/26, 10.1.2.144/29 e 10.1.2.156/30), com a de ENL2 em ECMP.

![show ip route no MTZ-R2 depois do OSPF](pics/mtz-r2-show-ip-route_depois%20ospf.png)

| | Antes | Depois |
| :--- | :--- | :--- |
| Sub-redes na tabela | 6 | 9 |
| Máscaras diferentes | 3 (/27, /30, /32) | 5 (/26, /27, /29, /30, /32) |
| Rotas `O` | nenhuma | 10.1.2.0/26, 10.1.2.144/29, 10.1.2.156/30 |

### Rotas OSPF no MTZ-R2
```
Router#show ip route ospf
     10.0.0.0/8 is variably subnetted, 9 subnets, 5 masks
O       10.1.2.0 [110/2] via 10.1.2.162, 00:06:32, GigabitEthernet0/2
O       10.1.2.144 [110/2] via 10.1.2.153, 00:10:49, GigabitEthernet0/1
O       10.1.2.156 [110/2] via 10.1.2.153, 00:06:32, GigabitEthernet0/1
                   [110/2] via 10.1.2.162, 00:06:32, GigabitEthernet0/2
```
- LAN-ADM (10.1.2.0/26): via MTZ-R3 (10.1.2.162), custo 2 (1 do enlace ENL3 + 1 da interface LAN do MTZ-R3).
- LAN-SRV (10.1.2.144/29): via MTZ-R1 (10.1.2.153), custo 2.
- ENL2 (10.1.2.156/30): **duas rotas de custo igual** (ECMP), uma via MTZ-R1 e outra via MTZ-R3. O MTZ-R2 distribui o tráfego entre as duas.
- `[110/2]`: 110 é a distância administrativa do OSPF, 2 é o custo. É o tema da Q5.

## Para forçar uma nova eleição (resposta da Q4)
O DR só muda se a adjacência cair e for refeita. Duas formas:
- `clear ip ospf process` nos roteadores do enlace. Nesse caso, ao reiniciarem juntos, o maior router-id vence.
- Subir a prioridade de um roteador com `ip ospf priority N` na interface, seguido de `clear ip ospf process`. Prioridade 0 impede o roteador de ser eleito.

## Quando o router-id é aplicado
- O `router-id` precisa ser definido antes do primeiro `network`.
- Se o processo já estava ativo, o novo ID só vale depois de `clear ip ospf process` nos dois lados do enlace.

## Evidências a coletar para a Q4
- `show ip ospf neighbor` no MTZ-R1 (colunas Neighbor ID, Pri, State, Dead Time, Address, Interface).
- `show ip ospf interface g0/1` no MTZ-R1 (Network Type, prioridade, DR e BDR). Repetir nos outros enlaces do triângulo.
- Esta ordem de configuração (R1, R2, R3).

## Comandos aplicados

```
! MTZ-R1
router ospf 1
 router-id 1.1.1.1
 network 10.1.2.144 0.0.0.7 area 0
 network 10.1.2.152 0.0.0.3 area 0
 network 10.1.2.156 0.0.0.3 area 0
 passive-interface g0/0

! MTZ-R2
router ospf 1
 router-id 1.1.1.2
 network 10.1.2.64 0.0.0.31 area 0
 network 10.1.2.152 0.0.0.3 area 0
 network 10.1.2.160 0.0.0.3 area 0
 passive-interface g0/0

! MTZ-R3
router ospf 1
 router-id 1.1.1.3
 network 10.1.2.0 0.0.0.63 area 0
 network 10.1.2.156 0.0.0.3 area 0
 network 10.1.2.160 0.0.0.3 area 0
 passive-interface g0/0
```
