# Contexto do trabalho (Grau A, Redes, Unisinos)

Resumo para retomar o trabalho em novas sessões. O enunciado completo está em `docs/Enunciado GA_*.pdf` e vale sobre este resumo.

## Visão geral
- **Tema:** integração de domínios de roteamento (OSPF × RIPv2) com DHCP centralizado. A InovaTech (matriz, OSPF) adquire a Legado (filial, RIPv2).
- **Ferramenta:** Cisco Packet Tracer 9.0. Roteadores Cisco 2911 com HWIC-2T. Módulos seriais são inseridos com o roteador desligado, antes de configurar. Desligar um roteador descarta a configuração não salva, então rode `copy running-config startup-config` antes.
- **Peso:** 3,0 pontos. Grupos de 4 a 6 alunos. Prazo original: 01/10/2026.
- **Avaliação:** simulações 40%, questionário 50%, plano de endereçamento 10%. O bônus (etapa 4) vale até 0,3.

## Etapas (um .pkt por etapa, no estado final correto)
1. **Matriz** (`etapa1.pkt`): MTZ-R1/R2/R3 em triângulo GigE (cabo cruzado), OSPF processo 1 área 0, DHCP centralizado no SRV-DC com relay (`ip helper-address`) em MTZ-R2 e MTZ-R3, HTTP no servidor. Questões Q1 a Q6.
2. **Filial** (`etapa2.pkt`): FIL-R1 e FIL-R2 com RIPv2 (`version 2`, `no auto-summary`, `network 10.0.0.0`), isolada da matriz, PCs com IP estático provisório. A matriz não muda em relação à etapa 1. Questões Q7 a Q10.
3. **Integração** (`etapa3.pkt`): R-BORDA com OSPF (só nas interfaces da matriz) e RIP (só na interface da filial), redistribuição mútua, pools e relay para as LANs da filial, PCs da filial em DHCP, teste de falha de serial. Questões Q11 a Q17.
4. **Bônus** (`etapa4_bonus.pkt`): filial migrada para OSPF área 1, R-BORDA vira ABR. Questão Q18.

## Convenções obrigatórias
- Hostnames exatos: MTZ-R1, MTZ-R2, MTZ-R3, R-BORDA, FIL-R1, FIL-R2, PC-ENG-1, PC-FIL2-1, SRV-DC etc.
- Gateway de cada LAN = primeiro IP útil. Servidor = segundo IP útil da LAN-SRV (estático).
- `router-id` 1.1.1.N antes do primeiro `network`: N=1..3 MTZ-R1..R3, 4 R-BORDA, 5 e 6 FIL-R1 e FIL-R2 (etapa 4). Se o processo já estava ativo, `clear ip ospf process` nos dois lados do enlace.
- Comandos `network` do OSPF sempre com wildcard exato da sub-rede. Nunca `10.0.0.0 0.255.255.255`.
- Seriais: o cabo DCE sai do roteador de nome alfabeticamente menor, com `clock rate 1544000` e `no shutdown` nas duas pontas. Confirmar up/up com `show ip interface brief`.
- Pools DHCP: *Maximum Number of Users* = IPs úteis menos o gateway. DNS vazio. Servidor precisa de gateway na aba Config.
- Evidências: saídas de CLI e Command Prompt em texto colado, com o prompt. Simulation em captura de tela com a coluna Time visível. HTTP por captura do Web Browser (`etapa1_http.png`, `etapa3_http.png`). Antes de `ipconfig /renew`, rodar `ipconfig /release`.

## Plano de endereçamento (bloco 10.1.0.0/22, /24 escolhido: 10.1.2.0/24)
Detalhes completos em `tabelaEnderecos.md` (sub-redes) e `enderecosRoteadores.md` (por roteador).

| Sub-rede | Rede | Gateway |
| :--- | :--- | :--- |
| LAN-ADM /26 | 10.1.2.0 | 10.1.2.1 |
| LAN-ENG /27 | 10.1.2.64 | 10.1.2.65 |
| LAN-FIL1 /27 | 10.1.2.96 | 10.1.2.97 |
| LAN-FIL2 /28 | 10.1.2.128 | 10.1.2.129 |
| LAN-SRV /29 | 10.1.2.144 | 10.1.2.145 (SRV-DC = .146) |
| ENL1 a ENL7 /30 | 10.1.2.152 a 10.1.2.176 | ver `enderecosRoteadores.md` |

Mapeamento dos enlaces: ENL1 R1–R2, ENL2 R1–R3, ENL3 R2–R3, ENL4 R2–BORDA, ENL5 R3–BORDA, ENL6 BORDA–FIL-R1, ENL7 FIL-R1–FIL-R2.

Interfaces (escolha do grupo): LAN em g0/0 nos roteadores com LAN. Seriais em `s0/3/x`, com `s0/2/0` no R-BORDA para o FIL-R1.

## Entrega (um .zip no Moodle)
- `respostas.md`: nomes e matrículas no início, depois Q1 a Q17 (+Q18). Cada resposta cita sua evidência.
- `plano_enderecamento.md`: tabela VLSM justificada (sub-rede, máscara, faixa útil, broadcast, atribuição, justificativa). O /24 escolhido deve constar da primeira linha.
- `evidencias/`: `etapa1.pdf`, `etapa2.pdf`, `etapa3.pdf` (e `etapa4.pdf`), `prints/` (Simulation, `topologia.png`, `etapa1_http.png`, `etapa3_http.png`), `configs/etapaN_ROTEADOR.txt` (14 arquivos: 3 na etapa 1, 5 na 2, 6 na 3).
- `pkt/`: `etapa1.pkt`, `etapa2.pkt`, `etapa3.pkt` (e `etapa4_bonus.pkt`).
- Os templates das evidências ficam no Moodle.

## Arquivos do repositório
- `tabelaEnderecos.md`: tabela VLSM.
- `enderecosRoteadores.md`: IPs por interface, OSPF, relay, DCE, RIP e redistribuição.
- `docs/`: enunciado e este contexto.

## Estado atual (2026-10-07)
- IPs, máscaras e clock rate configurados nos 6 roteadores, tudo no mesmo arquivo do Packet Tracer.
- Ainda não feito: OSPF, SRV-DC (IP, DHCP, HTTP), relay, PCs em DHCP e verificações da etapa 1.
- **Atenção:** o `etapa1.pkt` deve conter só a matriz. Salvar a topologia completa à parte, remover R-BORDA, FIL-R1, FIL-R2 e as LANs da filial da etapa 1, e tirar IP e clock das seriais de MTZ-R2 e MTZ-R3 (manter o HWIC-2T instalado). Reaplicar na etapa 3.
- O prazo original (01/10/2026) já passou. Confirmar com o professor.
