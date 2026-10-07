# Etapa 1 (matriz): estado e passagem de bastão

Leia primeiro `docs/contexto.md` (visão geral do trabalho) e o enunciado em `docs/Enunciado GA_*.pdf`. Este arquivo registra onde a Etapa 1 parou, o que já foi coletado e o que falta.

## Resumo do estado

| Item | Situação |
| :--- | :--- |
| Endereçamento dos 3 roteadores da matriz | feito |
| OSPF (processo 1, área 0) em MTZ-R1, R2, R3 | feito, vizinhanças FULL |
| SRV-DC: IP estático, gateway, pools DHCP, HTTP | feito |
| `ip helper-address` em MTZ-R2 e MTZ-R3 | feito (o DHCP funcionou nas duas LANs) |
| PCs em DHCP | PC-ENG-1 e PC-ADM-1 confirmados. Conferir PC-ENG-2 e PC-ADM-2 |
| Evidências de CLI e testes fim a fim | coletadas (abaixo), faltam algumas |
| Evidências em modo Simulation (Q1, Q2, Q3, Q6) | **não feitas** |
| Respostas Q1 a Q6 | **não escritas** |
| `evidencias/configs/etapa1_*.txt` | **não gerados** |
| `etapa1.pkt` final salvo em `pkt/` | **não feito** |

Os arquivos `etapa1.pkt` e `topologia-base.pkt` estão na raiz do repositório. `topologia-base.pkt` é a rede completa (matriz, borda e filial) usada para entender o programa e serve de base para as etapas 2 e 3. `etapa1.pkt` só tem a matriz.

## Topologia da Etapa 1
Três roteadores (MTZ-R1, R2, R3) em triângulo GigabitEthernet com cabo cruzado, três switches (LAN-SRV, LAN-ENG, LAN-ADM), SRV-DC e dois PCs por LAN (PC-ENG-1/2, PC-ADM-1/2).
- Roteador–switch com cabo direto.
- As seriais de MTZ-R2 e MTZ-R3 (`s0/3/0`) ficam sem IP, sem clock rate e em shutdown. O módulo HWIC-2T permanece instalado. Elas só voltam a ser usadas na Etapa 3.
- R-BORDA, FIL-R1, FIL-R2 e as LANs da filial não existem em `etapa1.pkt`.

## Endereçamento (resumo)
Detalhes completos em `tabelaEnderecos.md` e `enderecosRoteadores.md`.

| Roteador | g0/0 | g0/1 | g0/2 |
| :--- | :--- | :--- | :--- |
| MTZ-R1 | 10.1.2.145/29 (LAN-SRV) | 10.1.2.153/30 (ENL1, p/ R2) | 10.1.2.157/30 (ENL2, p/ R3) |
| MTZ-R2 | 10.1.2.65/27 (LAN-ENG) | 10.1.2.154/30 (ENL1, p/ R1) | 10.1.2.161/30 (ENL3, p/ R3) |
| MTZ-R3 | 10.1.2.1/26 (LAN-ADM) | 10.1.2.158/30 (ENL2, p/ R1) | 10.1.2.162/30 (ENL3, p/ R2) |

SRV-DC: 10.1.2.146/29, gateway 10.1.2.145, DNS vazio.

## OSPF

| Roteador | router-id | `network` (área 0) | passive-interface |
| :--- | :--- | :--- | :--- |
| MTZ-R1 | 1.1.1.1 | 10.1.2.144 0.0.0.7, 10.1.2.152 0.0.0.3, 10.1.2.156 0.0.0.3 | g0/0 |
| MTZ-R2 | 1.1.1.2 | 10.1.2.64 0.0.0.31, 10.1.2.152 0.0.0.3, 10.1.2.160 0.0.0.3 | g0/0 |
| MTZ-R3 | 1.1.1.3 | 10.1.2.0 0.0.0.63, 10.1.2.156 0.0.0.3, 10.1.2.160 0.0.0.3 | g0/0 |

Ordem de configuração: R1, depois R2, depois R3. Isso importa para a Q4. Ver `docs/ordem-configuracao-ospf.md`.

## DHCP no SRV-DC (Services → DHCP)
O pool padrão `serverPool` foi removido.

| Campo | LAN-ENG | LAN-ADM |
| :--- | :--- | :--- |
| Gateway | 10.1.2.65 | 10.1.2.1 |
| DNS | vazio | vazio |
| Start IP | 10.1.2.66 | 10.1.2.2 |
| Máscara | 255.255.255.224 | 255.255.255.192 |
| Max. users | 29 | 61 |

Regra: Max. users = IPs úteis menos o gateway. Relay: `ip helper-address 10.1.2.146` na g0/0 de MTZ-R2 e MTZ-R3.

---

## Evidências já geradas

### 1. `ping` do SRV-DC para a interface LAN do MTZ-R2 (obrigatória)
`TTL=254`: a resposta saiu do MTZ-R2 com 255 e passou por um roteador (MTZ-R1).
```
C:\>ping 10.1.2.65

Pinging 10.1.2.65 with 32 bytes of data:

Reply from 10.1.2.65: bytes=32 time<1ms TTL=254
Reply from 10.1.2.65: bytes=32 time<1ms TTL=254
Reply from 10.1.2.65: bytes=32 time<1ms TTL=254
Reply from 10.1.2.65: bytes=32 time<1ms TTL=254

Ping statistics for 10.1.2.65:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

### 2. `show ip ospf neighbor` no MTZ-R1 (obrigatória)
Dois vizinhos FULL. O estado descreve o papel do vizinho, então o MTZ-R1 é DR nos dois enlaces.
```
Router#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           1   FULL/BDR        00:00:35    10.1.2.154      GigabitEthernet0/1
1.1.1.3           1   FULL/BDR        00:00:32    10.1.2.158      GigabitEthernet0/2
```

### 3. `show ip route` no MTZ-R2, antes do OSPF
Só `C` e `L`, 6 sub-redes em 3 máscaras. Print: `docs/pics/mtz-r2-show-ip-route_antes ospf.png`.
```
Router>show ip route
Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 6 subnets, 3 masks
C       10.1.2.64/27 is directly connected, GigabitEthernet0/0
L       10.1.2.65/32 is directly connected, GigabitEthernet0/0
C       10.1.2.152/30 is directly connected, GigabitEthernet0/1
L       10.1.2.154/32 is directly connected, GigabitEthernet0/1
C       10.1.2.160/30 is directly connected, GigabitEthernet0/2
L       10.1.2.161/32 is directly connected, GigabitEthernet0/2
```

### 4. `show ip route` no MTZ-R2, depois do OSPF (obrigatória)
Transcrito do print `docs/pics/mtz-r2-show-ip-route_depois ospf.png`. A rota para o ENL2 (10.1.2.156/30) tem custo igual por dois caminhos (ECMP).
```
Router#show ip route
Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 9 subnets, 5 masks
O       10.1.2.0/26 [110/2] via 10.1.2.162, 00:05:05, GigabitEthernet0/2
C       10.1.2.64/27 is directly connected, GigabitEthernet0/0
L       10.1.2.65/32 is directly connected, GigabitEthernet0/0
O       10.1.2.144/29 [110/2] via 10.1.2.153, 00:09:22, GigabitEthernet0/1
C       10.1.2.152/30 is directly connected, GigabitEthernet0/1
L       10.1.2.154/32 is directly connected, GigabitEthernet0/1
O       10.1.2.156/30 [110/2] via 10.1.2.153, 00:05:05, GigabitEthernet0/1
                      [110/2] via 10.1.2.162, 00:05:05, GigabitEthernet0/2
C       10.1.2.160/30 is directly connected, GigabitEthernet0/2
L       10.1.2.161/32 is directly connected, GigabitEthernet0/2
```
Observação: o texto colado na conversa não trazia as máscaras (/26, /29, /30) nas linhas `O`. As máscaras acima vêm do print. Para a entrega, rode o comando de novo e cole o texto completo.

### 5. `show ip protocols` no MTZ-R1
```
Routing Protocol is "ospf 1"
  Outgoing update filter list for all interfaces is not set 
  Incoming update filter list for all interfaces is not set 
  Router ID 1.1.1.1
  Number of areas in this router is 1. 1 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
    10.1.2.144 0.0.0.7 area 0
    10.1.2.152 0.0.0.3 area 0
    10.1.2.156 0.0.0.3 area 0
  Passive Interface(s): 
    GigabitEthernet0/0
  Routing Information Sources:  
    Gateway         Distance      Last Update 
    1.1.1.1              110      00:00:43
  Distance: (default is 110)
```

### 6. `show ip ospf interface g0/1` no MTZ-R1 (Q4, Q5, Q6)
```
Router#show ip ospf interface g0/1

GigabitEthernet0/1 is up, line protocol is up
  Internet address is 10.1.2.153/30, Area 0
  Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 1.1.1.1, Interface address 10.1.2.153
  Backup Designated Router (ID) 1.1.1.2, Interface address 10.1.2.154
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Hello due in 00:00:08
  Index 2/2, flood queue length 0
  Next 0x0(0)/0x0(0)
  Last flood scan length is 1, maximum is 1
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 1.1.1.2  (Backup Designated Router)
  Suppress hello for 0 neighbor(s)
```

### 7. `show ip route ospf` no MTZ-R2 (Q5)
Sem as máscaras, pelo mesmo motivo do item 4.
```
Router#show ip route ospf
     10.0.0.0/8 is variably subnetted, 9 subnets, 5 masks
O       10.1.2.0 [110/2] via 10.1.2.162, 00:06:32, GigabitEthernet0/2
O       10.1.2.144 [110/2] via 10.1.2.153, 00:10:49, GigabitEthernet0/1
O       10.1.2.156 [110/2] via 10.1.2.153, 00:06:32, GigabitEthernet0/1
                   [110/2] via 10.1.2.162, 00:06:32, GigabitEthernet0/2
```

### 8. `ipconfig /all` em PC-ENG-1 e PC-ADM-1 (obrigatória)
Endereço via DHCP, dentro da faixa e com o gateway certo. O servidor DHCP (10.1.2.146) está em outra sub-rede, o que mostra o relay funcionando. O bloco "Bluetooth Connection" é uma interface virtual do Packet Tracer, sem uso.

PC-ENG-1:
```
C:\>ipconfig /all

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: 
   Physical Address................: 0001.C96B.B973
   Link-local IPv6 Address.........: FE80::201:C9FF:FE6B:B973
   IPv6 Address....................: ::
   IPv4 Address....................: 10.1.2.66
   Subnet Mask.....................: 255.255.255.224
   Default Gateway.................: ::
                                     10.1.2.65
   DHCP Servers....................: 10.1.2.146
   DHCPv6 IAID.....................: 
   DHCPv6 Client DUID..............: 00-01-00-01-CE-22-55-8E-00-01-C9-6B-B9-73
   DNS Servers.....................: ::
                                     0.0.0.0
```

PC-ADM-1:
```
C:\>ipconfig /all

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: 
   Physical Address................: 00D0.5894.2955
   Link-local IPv6 Address.........: FE80::2D0:58FF:FE94:2955
   IPv6 Address....................: ::
   IPv4 Address....................: 10.1.2.2
   Subnet Mask.....................: 255.255.255.192
   Default Gateway.................: ::
                                     10.1.2.1
   DHCP Servers....................: 10.1.2.146
   DHCPv6 IAID.....................: 
   DHCPv6 Client DUID..............: 00-01-00-01-CE-22-55-8E-00-D0-58-94-29-55
   DNS Servers.....................: ::
                                     0.0.0.0
```

### 9. `ping` de PC-ENG-1 para PC-ADM-1 (obrigatória)
O primeiro `Request timed out` é a resolução ARP do gateway. `TTL=126`: o PC-ADM-1 responde com 128 e a resposta passa por dois roteadores (MTZ-R3 e MTZ-R2), ou seja, pelo enlace direto ENL3.
```
C:\>ping 10.1.2.2

Pinging 10.1.2.2 with 32 bytes of data:

Request timed out.
Reply from 10.1.2.2: bytes=32 time<1ms TTL=126
Reply from 10.1.2.2: bytes=32 time<1ms TTL=126
Reply from 10.1.2.2: bytes=32 time<1ms TTL=126

Ping statistics for 10.1.2.2:
    Packets: Sent = 4, Received = 3, Lost = 1 (25% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

### 10. Teste HTTP (obrigatória)
Captura do Web Browser do PC-ENG-1 em `http://10.1.2.146`, com a página padrão do Packet Tracer. Arquivo: `docs/pics/etapa1_http.png`. Na entrega vai para `evidencias/prints/etapa1_http.png`.

### Arquivos de imagem existentes
- `docs/pics/mtz-r2-show-ip-route_antes ospf.png`
- `docs/pics/mtz-r2-show-ip-route_depois ospf.png`
- `docs/pics/etapa1_http.png`

---

## O que falta fazer

### A. Evidências ainda não coletadas
- [ ] `show ip route` no MTZ-R2, saída atual completa, só como texto (já há uma versão no item 4, mas vale regerar no estado final).
- [ ] `show ip interface brief` nos 3 roteadores.
- [ ] `show ip ospf interface g0/2` no MTZ-R1 e `show ip ospf interface g0/2` no MTZ-R2 (DR/BDR do ENL3, para completar a tabela da Q4 em `docs/ordem-configuracao-ospf.md`).
- [ ] `show ip protocols` no MTZ-R2 e MTZ-R3 (opcional).
- [ ] Conferir PC-ENG-2 e PC-ADM-2 em DHCP.
- [ ] Capturas em modo Simulation (ver Q1, Q2, Q3, Q6 abaixo). Devem mostrar a coluna *Time*.
- [ ] Topologia completa com rótulos legíveis: `evidencias/prints/topologia.png`.

### B. Questões Q1 a Q6 (respostas em `respostas.md`, cada uma citando a evidência)
Nenhuma foi escrita ainda.

| Questão | O que fazer | Evidência |
| :--- | :--- | :--- |
| Q1 | Simulation, filtro só DHCP. `ipconfig /release` e depois `ipconfig /renew` no PC-ENG-1. Tabela com as 4 mensagens (DISCOVER, OFFER, REQUEST, ACK): IP origem, IP destino, MAC destino. Usar só a versão observada na LAN do cliente. Explicar por que o cliente sai de 0.0.0.0. | capturas de PDU |
| Q2 | Comparar o DISCOVER antes e depois de passar pelo MTZ-R2 (Inbound x Outbound PDU Details): o que muda nos IPs, e o campo GIADDR. | capturas de PDU |
| Q3 | `no ip helper-address` na g0/0 do MTZ-R2, `ipconfig /renew` no PC, ver o DISCOVER morrer no MTZ-R2 (aba OSI Model). Explicar por que broadcast não atravessa roteador. **Repor o helper-address depois.** | captura da Simulation |
| Q4 | `show ip ospf neighbor` no MTZ-R1, explicar colunas, DR/BDR em enlace com 2 roteadores, comparar o DR eleito com os router-id. Ver `docs/ordem-configuracao-ospf.md`. Falta confirmar o ENL3. | itens 2 e 6 + ENL3 |
| Q5 | Decompor `[110/2]` da rota para a LAN-SRV no MTZ-R2, mostrar a conta do custo e explicar a referência de 100 Mb/s (FastEthernet teria o mesmo custo 1 do GigE). | itens 4 e 6 |
| Q6 | Intervalos Hello (10) e Dead (40). Medir o Hello no modo Simulation (filtro só OSPF, diferença de *Time* entre dois Hellos consecutivos). | item 6 + captura |

### C. Fechamento da etapa
- [ ] Gerar `evidencias/configs/etapa1_MTZ-R1.txt`, `etapa1_MTZ-R2.txt`, `etapa1_MTZ-R3.txt` com `terminal length 0` e `show running-config` em cada roteador. Fazer **depois** de concluir as questões e repor qualquer experimento (Q3).
- [ ] Salvar o `etapa1.pkt` final no estado correto e copiar para `pkt/etapa1.pkt`.
- [ ] Gerar `evidencias/etapa1.pdf` a partir do template da etapa 1 no Moodle, com as seções fixas preenchidas.
- [ ] Preencher `plano_enderecamento.md` (tabela VLSM justificada). O `tabelaEnderecos.md` não tem as colunas de atribuição e justificativa.
- [ ] Nomes e matrículas no início do `respostas.md`.

## Pontos de atenção
- **Estado final do .pkt:** o enunciado quer cada `.pkt` e cada config no estado final correto, com os experimentos revertidos. Em especial, repor o `ip helper-address` do MTZ-R2 depois da Q3.
- **Salvar a configuração:** ao desligar um roteador no Packet Tracer, a configuração não salva se perde. Rodar `copy running-config startup-config` antes de mexer em módulos ou desligar.
- **ipconfig /renew:** o enunciado pede `ipconfig /release` antes, no mesmo PC.
- **Evidências de CLI:** texto colado com o prompt e a linha do comando, nunca captura de tela. Simulation: captura com a coluna *Time*. HTTP: captura do Web Browser.
- **Prazo:** a data original (01/10/2026) já passou. Confirmar com o professor.
- **Etapa 2 (próxima):** partir de uma cópia do `etapa1.pkt` e **não alterar a matriz**. Acrescentar FIL-R1, FIL-R2, as LANs da filial e RIPv2 (valores em `enderecosRoteadores.md`).
