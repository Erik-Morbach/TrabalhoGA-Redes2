# Trabalho do Grau A, Redes (Unisinos): OSPF × RIPv2 com DHCP centralizado

Projeto de rede no Cisco Packet Tracer 9.0. O repositório guarda os `.pkt`, a documentação e as evidências. O Claude não abre o Packet Tracer: os comandos e as capturas são executados pelo aluno no programa.

## Leia primeiro
1. `docs/contexto.md`: visão geral do trabalho, etapas, convenções e estrutura da entrega.
2. `etapa1.md`: estado atual da Etapa 1, evidências já coletadas e checklist do que falta.
3. `docs/Enunciado GA_*.pdf`: enunciado completo. Vale mais que qualquer resumo.
4. `tabelaEnderecos.md` e `enderecosRoteadores.md`: plano de endereçamento e configuração por roteador.
5. `docs/ordem-configuracao-ospf.md`: ordem de configuração do OSPF e análise de DR/BDR (Q4).

## Onde o trabalho parou
Etapa 1 (matriz): endereçamento, OSPF, DHCP no SRV-DC, relay e PCs em DHCP já funcionam, e várias evidências de CLI estão coletadas em `etapa1.md`.

**Próximo passo:** as evidências em modo Simulation e as respostas Q1 a Q6. Comece pela **Q1** (conversa DHCP completa do PC-ENG-1):
1. `ipconfig /release` no PC-ENG-1.
2. Modo Simulation, filtro só DHCP (Edit Filters).
3. `ipconfig /renew` e avançar passo a passo.
4. Anotar, para DISCOVER, OFFER, REQUEST e ACK na LAN do cliente: IP de origem, IP de destino e MAC de destino.

Depois: Q2, Q3, Q4 (falta confirmar DR/BDR do ENL3), Q5 e Q6. Em seguida, o fechamento da etapa (configs, `.pkt`, PDF) descrito em `etapa1.md`.

## Regras que o enunciado impõe (não esquecer)
- Hostnames exatos: MTZ-R1, MTZ-R2, MTZ-R3, R-BORDA, FIL-R1, FIL-R2, PC-ENG-1, SRV-DC etc.
- Gateway de cada LAN = primeiro IP útil. Servidor = segundo IP útil da LAN-SRV.
- OSPF processo 1, área 0. `router-id` 1.1.1.N fixado **antes** do primeiro `network` (N=1..3 nas MTZ, 4 no R-BORDA, 5 e 6 nas FIL-R1 e FIL-R2 só no bônus). Se o processo já estava ativo, `clear ip ospf process` nos dois lados do enlace.
- Todo `network` do OSPF com **wildcard exato** da sub-rede. Nunca `10.0.0.0 0.255.255.255`.
- Seriais: o cabo DCE sai do roteador de nome alfabeticamente menor do par, com `clock rate 1544000`. Confirmar up/up com `show ip interface brief`.
- Pools DHCP: *Maximum Number of Users* = IPs úteis menos o gateway. DNS vazio.
- Evidências de CLI e Command Prompt: **texto colado**, com prompt e comando. Simulation: **captura de tela** com a coluna *Time*. HTTP: captura do Web Browser.
- Antes de `ipconfig /renew`, rodar `ipconfig /release` no mesmo PC.
- Cada `.pkt` e cada config entregue no **estado final correto**, com experimentos revertidos (Q3: repor o `ip helper-address`; Q12: `redistribute ospf 1 metric 3`; Q16: serial em `no shutdown`).
- No Packet Tracer, desligar um roteador descarta a configuração não salva. Antes de inserir módulo em roteador configurado, `copy running-config startup-config`.
- A matriz da Etapa 1 **não pode mudar** na Etapa 2. Na Etapa 3, só são permitidas as alterações listadas no enunciado em MTZ-R2 e MTZ-R3 (serial para R-BORDA, `network` do /30 no OSPF e os novos pools no SRV-DC).

## Arquivos do repositório

| Arquivo | Conteúdo |
| :--- | :--- |
| `etapa1.pkt` | Packet Tracer da Etapa 1 (só a matriz) |
| `topologia-base.pkt` | rede completa (matriz, borda e filial), usada como base para as etapas 2 e 3 |
| `etapa1.md` | estado da Etapa 1, evidências e pendências |
| `tabelaEnderecos.md` | sub-redes (VLSM) |
| `enderecosRoteadores.md` | IPs por interface, OSPF, relay, DCE, RIP e redistribuição |
| `docs/` | enunciado, contexto, ordem do OSPF e `pics/` com capturas |

## Estrutura da entrega (um .zip no Moodle)
`respostas.md` (nomes e matrículas no início, Q1 a Q17 e Q18 se houver bônus), `plano_enderecamento.md`, `evidencias/` (`etapaN.pdf`, `prints/`, `configs/etapaN_ROTEADOR.txt`) e `pkt/` (`etapa1.pkt`, `etapa2.pkt`, `etapa3.pkt`, e `etapa4_bonus.pkt` se houver). Os nomes são fixos. O prazo original (01/10/2026) já passou, então confirmar com o professor.
