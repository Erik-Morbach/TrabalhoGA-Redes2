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

Q7. Os quatro relógios do RIP. Cole o show ip protocols de FIL-R1 e extraia os valores dos ti‐
mers de update, invalid, holddown e flush. Explique em uma frase o papel de cada um e o
que significa a linha "next update due in...".
Onde procurar: CLI de FIL-R1, primeiro bloco da saída de show ip protocols.
Q8. Por dentro de um Response. No modo Simulation, capture um RIP Response periódico
de FIL-R1. Para qual endereço de destino ele é enviado, e por que RIPv2 usa multicast em
vez do broadcast do RIPv1? Liste os campos presentes em cada entrada de rota do Respon‐
se e diga qual deles é a razão de o RIPv2 suportar VLSM.
Onde procurar: modo Simulation com filtro só em RIP. Outbound PDU Details do Response (destino
224.0.0.9, procure o campo de máscara nas entradas). Se os PDU Details da sua versão não exibirem o
campo de máscara nas entradas de rota, use como evidência o show ip route de FIL-R2, que mostra subredes de tamanhos diferentes convivendo dentro da mesma rede 10.0.0.0, e explique por que isso seria
impossível com RIPv1.
Q9. Métrica em saltos. Na tabela de FIL-R2, interprete a rota R para a LAN-FIL1: o que signifi‐
ca [120/1]? Por que a métrica do RIP é contada em saltos, e qual a consequência prática do
limite de 15?
Onde procurar: show ip route rip em FIL-R2.

Q10. Tagarelice comparada. Com as duas redes estáveis (sem qualquer mudança), compa‐
re o que RIP e OSPF continuam transmitindo periodicamente: o quê, com que frequência e
de que tamanho? Use os tamanhos reais dos pacotes observados para estimar quantos by‐
tes por minuto cada protocolo gasta em um enlace parado, e conclua qual dos dois cresce
quando a rede ganha novas sub-redes. Atenção: com o número pequeno de rotas desta to‐
pologia os dois valores medidos ficam próximos, e podem até inverter. A conclusão pedida
é sobre o comportamento de cada protocolo à medida que a tabela de rotas cresce, não so‐
bre qual número é maior nesta medição.
Onde procurar: modo Simulation, uma rodada com filtro OSPF (na matriz) e outra com filtro RIP (na
filial). O tamanho do pacote aparece nos PDU Details e a periodicidade, na coluna Time.

## Etapa 3

Q11. O network classful do RIP. Em R-BORDA, o comando network 10.0.0.0 sob router rip
ativa o RIP em quais interfaces? Por que isso é um problema em um roteador de borda cu‐
jas interfaces estão todas dentro de 10.0.0.0/8, e o que exatamente o passive-interface
contém, ou seja, o que ele deixa de enviar e o que continua valendo? Cole a evidência de que
os updates RIP saem apenas pela interface correta.
Onde procurar: show ip protocols em R-BORDA: o bloco do RIP lista as interfaces em que ele está ativo e
as interfaces passivas.
Q12. A métrica-semente que faltou. Configure primeiro redistribute ospf 1 metric 16 no
router rip de R-BORDA. As rotas da matriz aparecem em FIL-R1? Documente o sintoma
(show ip route antes), explique a causa, ou seja, o que a métrica 16 significa para o RIP, e
corrija com metric 3, documentando o resultado (show ip route depois). Explique também
por que omitir o parâmetro metric produziria o mesmo efeito em um roteador Cisco real.
Onde procurar: show ip route em FIL-R1 antes e depois. O bloco Redistributing do show ip protocols em
R-BORDA confirma que a redistribuição está ativa nos dois casos, o que isola a métrica como causa
do sintoma.
Q13. Rotas de segunda mão no OSPF. Como as redes da filial aparecem na tabela de MTZR1? Explique o código da rota e a notação [110/20]: de onde vem o 20, o que significa o "E2"
e, comparando a mesma rota em MTZ-R1 e MTZ-R3, o que acontece (ou não acontece) com
essa métrica ao longo do caminho?
Onde procurar: show ip route em MTZ-R1 e MTZ-R3 (rotas O E2).
Q14. Rotas de segunda mão no RIP. Como a LAN-SRV aparece na tabela de FIL-R2? Decom‐
ponha a métrica observada em (métrica-semente + saltos dentro do domínio RIP). O que
aconteceria com essa rota se houvesse mais 13 roteadores em cadeia na filial?
Onde procurar: show ip route rip em FIL-R2, comparado com o de FIL-R1.

Q15. A jornada completa de um DISCOVER. Converta PC-FIL2-1 para DHCP e capture a ob‐
tenção de endereço no modo Simulation. Liste, em ordem, todos os dispositivos que o DIS‐
COVER (e sua versão encaminhada) atravessa até o SRV-DC, e o caminho de volta do OF‐
FER. Em cada trecho, diga qual protocolo de roteamento aprendeu a rota que está sendo
usada, e responda: o roteador encaminha o pacote consultando o protocolo ou a tabela de
rotas? Quantos roteadores separam o cliente do servidor DHCP, e por que isso não impede
o serviço?
Onde procurar: Event List (coluna At Device) com filtro DHCP. Use show ip route nos roteadores do cami‐
nho para classificar cada rota (R, O, O E2, C).
Q16. Falha com rede viva. Em R-BORDA, cole o trecho do show ip route que mostra duas ro‐
tas de custo igual para a LAN-SRV e explique por que elas existem. Faça o tracert de PCFIL2-1 ao SRV-DC e identifique na saída por qual das duas seriais de R-BORDA o caminho
saiu. Derrube essa serial com shutdown, repita o tracert e cole os dois resultados. O cami‐
nho migrou para a outra serial, como esperado? Explique por que derrubar a serial que não
aparecia no primeiro tracert não produziria nenhuma mudança visível, embora as duas rotas
estejam na tabela. Com a serial ainda derrubada, execute ipconfig /renew em PC-FIL2-1 e
cole o resultado: o PC continua obtendo endereço? Explique por qual caminho o DISCOVER
e o OFFER trafegaram desta vez, e por que um serviço de DHCP centralizado a cinco rotea‐
dores de distância sobreviveu à falha de um enlace.
Onde procurar: show ip route em R-BORDA (duas linhas "via ..." sob o mesmo prefixo). O Command
Prompt do PC serve para os dois tracert e para o ipconfig /renew após a falha.
Q17. A fronteira não é simétrica. Na tabela de MTZ-R1, o enlace /30 entre R-BORDA e FIL-R1
não aparece. Na tabela de FIL-R2, os enlaces /30 entre R-BORDA e a matriz aparecem, e
com métrica menor do que as LANs da matriz. Explique as duas observações usando
três mecanismos distintos: o que a redistribuição carrega da tabela de rotas, o que o
comando network 10.0.0.0 coloca dentro do processo RIP, e o que exatamente o passiveinterface suprime.
Onde procurar: show ip route em MTZ-R1 e em FIL-R2. Em R-BORDA, compare o show ip route,
observando o código C dos três /30, com o bloco do RIP em show ip protocols, que lista as interfaces
ativas e as passivas.
