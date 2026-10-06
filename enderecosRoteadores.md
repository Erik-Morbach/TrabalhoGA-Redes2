## Mapeamento dos enlaces /30

| Enlace | Entre | Rede | Ponta A | Ponta B |
| :--- | :--- | :--- | :--- | :--- |
| ENL1 | MTZ-R1 ↔ MTZ-R2 | 10.1.2.152/30 | MTZ-R1 = .153 | MTZ-R2 = .154 |
| ENL2 | MTZ-R1 ↔ MTZ-R3 | 10.1.2.156/30 | MTZ-R1 = .157 | MTZ-R3 = .158 |
| ENL3 | MTZ-R2 ↔ MTZ-R3 | 10.1.2.160/30 | MTZ-R2 = .161 | MTZ-R3 = .162 |
| ENL4 | MTZ-R2 ↔ R-BORDA | 10.1.2.164/30 | MTZ-R2 = .165 | R-BORDA = .166 |
| ENL5 | MTZ-R3 ↔ R-BORDA | 10.1.2.168/30 | MTZ-R3 = .169 | R-BORDA = .170 |
| ENL6 | R-BORDA ↔ FIL-R1 | 10.1.2.172/30 | R-BORDA = .173 | FIL-R1 = .174 |
| ENL7 | FIL-R1 ↔ FIL-R2 | 10.1.2.176/30 | FIL-R1 = .177 | FIL-R2 = .178 |

ENL4 e ENL5 são seriais e entram na etapa 3. ENL6 e ENL7 entram nas etapas 2 e 3.

## Endereços por roteador

| Roteador | Interface | Liga em | IP | Máscara |
| :--- | :--- | :--- | :--- | :--- |
| **MTZ-R1** | g0/0 | LAN-SRV (gateway) | 10.1.2.145 | 255.255.255.248 |
| | g0/1 | MTZ-R2 (ENL1) | 10.1.2.153 | 255.255.255.252 |
| | g0/2 | MTZ-R3 (ENL2) | 10.1.2.157 | 255.255.255.252 |
| **MTZ-R2** | g0/0 | LAN-ENG (gateway) | 10.1.2.65 | 255.255.255.224 |
| | g0/1 | MTZ-R1 (ENL1) | 10.1.2.154 | 255.255.255.252 |
| | g0/2 | MTZ-R3 (ENL3) | 10.1.2.161 | 255.255.255.252 |
| | s0/3/0 | R-BORDA (ENL4, etapa 3) | 10.1.2.165 | 255.255.255.252 |
| **MTZ-R3** | g0/0 | LAN-ADM (gateway) | 10.1.2.1 | 255.255.255.192 |
| | g0/1 | MTZ-R1 (ENL2) | 10.1.2.158 | 255.255.255.252 |
| | g0/2 | MTZ-R2 (ENL3) | 10.1.2.162 | 255.255.255.252 |
| | s0/3/0 | R-BORDA (ENL5, etapa 3) | 10.1.2.169 | 255.255.255.252 |
| **R-BORDA** | s0/3/0 | MTZ-R2 (ENL4) | 10.1.2.166 | 255.255.255.252 |
| | s0/3/1 | MTZ-R3 (ENL5) | 10.1.2.170 | 255.255.255.252 |
| | s0/2/0 | FIL-R1 (ENL6) | 10.1.2.173 | 255.255.255.252 |
| **FIL-R1** | g0/0 | LAN-FIL1 (gateway) | 10.1.2.97 | 255.255.255.224 |
| | s0/3/0 | R-BORDA (ENL6) | 10.1.2.174 | 255.255.255.252 |
| | s0/3/1 | FIL-R2 (ENL7) | 10.1.2.177 | 255.255.255.252 |
| **FIL-R2** | g0/0 | LAN-FIL2 (gateway) | 10.1.2.129 | 255.255.255.240 |
| | s0/3/0 | FIL-R1 (ENL7) | 10.1.2.178 | 255.255.255.252 |

SRV-DC: 10.1.2.146/29, gateway 10.1.2.145.

## OSPF (processo 1, área 0) e relay

| Roteador | router-id | Comandos `network` | passive-interface | helper-address |
| :--- | :--- | :--- | :--- | :--- |
| MTZ-R1 | 1.1.1.1 | 10.1.2.144 0.0.0.7, 10.1.2.152 0.0.0.3, 10.1.2.156 0.0.0.3 | g0/0 | não precisa |
| MTZ-R2 | 1.1.1.2 | 10.1.2.64 0.0.0.31, 10.1.2.152 0.0.0.3, 10.1.2.160 0.0.0.3 | g0/0 | 10.1.2.146 na g0/0 |
| MTZ-R3 | 1.1.1.3 | 10.1.2.0 0.0.0.63, 10.1.2.156 0.0.0.3, 10.1.2.160 0.0.0.3 | g0/0 | 10.1.2.146 na g0/0 |

## Seriais: lado DCE (`clock rate 1544000`)

O DCE fica no roteador de nome alfabeticamente menor do par.

| Enlace | DCE | DTE |
| :--- | :--- | :--- |
| ENL4 | MTZ-R2 (s0/3/0) | R-BORDA (s0/3/0) |
| ENL5 | MTZ-R3 (s0/3/0) | R-BORDA (s0/3/1) |
| ENL6 | FIL-R1 (s0/3/0) | R-BORDA (s0/2/0) |
| ENL7 | FIL-R1 (s0/3/1) | FIL-R2 (s0/3/0) |

## Roteamento da filial e da borda

| Roteador | Etapa | Protocolo | Configuração |
| :--- | :---: | :--- | :--- |
| FIL-R1 | 2 | RIPv2 | `router rip`, `version 2`, `no auto-summary`, `network 10.0.0.0`, `passive-interface g0/0` |
| FIL-R2 | 2 | RIPv2 | igual ao FIL-R1 |
| FIL-R1, FIL-R2 | 3 | DHCP relay | `ip helper-address 10.1.2.146` na g0/0 |
| R-BORDA | 3 | OSPF 1, área 0 | router-id 1.1.1.4; `network 10.1.2.164 0.0.0.3 area 0`; `network 10.1.2.168 0.0.0.3 area 0`; `passive-interface s0/2/0` |
| R-BORDA | 3 | RIPv2 | `version 2`, `no auto-summary`, `network 10.0.0.0`; `passive-interface s0/3/0` e `s0/3/1` |
| R-BORDA | 3 | Redistribuição | `router ospf 1` → `redistribute rip subnets`; `router rip` → `redistribute ospf 1 metric 3` |
