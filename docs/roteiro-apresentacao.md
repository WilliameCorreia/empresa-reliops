# Roteiro de Apresentação — Postmortem INC-001
**Grupo ReliOps — Pós-graduação UNIFOR | Documentação Técnica | Prof. Arimatéia Júnior**
**Duração estimada: 18–20 minutos | 6 apresentadores**

---

## Divisão de Falas

| Apresentador | Seção | Tema | Tempo estimado |
|---|---|---|---|
| Pessoa 1 | Abertura | Contexto, empresa e o que é um postmortem | ~3 min |
| Pessoa 2 | Seções 1 e 2 | Resumo executivo e impacto no negócio | ~3 min |
| Pessoa 3 | Seções 3 e 4 | Linha do tempo e detecção | ~3 min |
| Pessoa 4 | Seção 5 | Causa raiz e técnica dos 5 Porquês | ~3 min |
| Pessoa 5 | Seções 6 e 7 | Mitigação, trade-offs e ações corretivas | ~3 min |
| Pessoa 6 | Seções 8, 9 e Governança | Lições aprendidas, acompanhamento e encerramento | ~3 min |

---

## Roteiro Completo

---

### PESSOA 1 — Abertura e Contexto (~3 min)

> *Apresenta o grupo, explica o contexto acadêmico e o que é um postmortem.*

---

"Bom dia a todos. Somos o grupo [nome do grupo], da turma de Documentação Técnica da pós-graduação da UNIFOR.

O trabalho que vamos apresentar hoje é um **postmortem técnico** — um tipo de documento muito usado em empresas de tecnologia para registrar formalmente o que aconteceu durante um incidente, por que aconteceu e o que vai ser feito para evitar que se repita.

A empresa do nosso cenário é a **ReliOps** — uma empresa fictícia que vende um sistema de observabilidade como serviço. Em termos simples: os clientes da ReliOps usam a plataforma para monitorar as próprias aplicações em tempo real, recebendo alertas e dashboards atualizados.

O nosso trabalho nasceu de uma situação real na ReliOps: um incidente que aconteceu, foi resolvido, mas **nunca foi registrado formalmente**. Sem linha do tempo, sem causa raiz documentada, sem ações com responsável e prazo definidos. O postmortem que vamos apresentar corrige exatamente esse gap.

Vou passar a palavra para [nome da Pessoa 2], que vai apresentar o resumo do incidente e o impacto que ele causou."

---

### PESSOA 2 — Resumo Executivo e Impacto (~3 min)

> *Apresenta o que aconteceu, os números do impacto e o diagrama de fluxo.*

---

"Obrigado, [nome da Pessoa 1].

O incidente aconteceu no dia **10 de março de 2026**. Um cliente — identificado no documento como T-2204 — ativou uma nova integração com a plataforma e, ao fazer isso, enviou um grande volume de dados históricos de uma só vez. O volume enviado foi **aproximadamente 4 vezes maior** do que o sistema estava preparado para receber.

Para entender o impacto, é importante entender como a plataforma funciona. [aponta para o diagrama no documento ou slide]

Em condições normais, os dados dos clientes chegam pela API de Ingestão, passam pelo Serviço de Processamento e chegam ao banco de dados — que alimenta os dashboards e alertas. Durante o incidente, o Serviço de Processamento não aguentou o volume e começou a acumular uma fila gigantesca de eventos esperando para ser processados.

O resultado foi direto: **dashboards congelados e alertas chegando com até 34 minutos de atraso** para cerca de **380 clientes**.

E aqui está um número importante para o impacto no contrato: o SLO que a ReliOps vende aos clientes é de 99,9% de disponibilidade por mês — o que equivale a **no máximo 44 minutos de problemas por mês**. Este incidente sozinho causou **156 minutos** de degradação — mais de **3 vezes o limite mensal** contratado.

Passo a palavra para [nome da Pessoa 3], que vai apresentar como o incidente evoluiu e como foi detectado."

---

### PESSOA 3 — Linha do Tempo e Detecção (~3 min)

> *Apresenta os eventos cronológicos e o tempo de detecção.*

---

"Obrigado, [nome da Pessoa 2].

A linha do tempo mostra como o incidente se desenvolveu minuto a minuto. Vou destacar os pontos mais relevantes.

O cliente T-2204 ativou a integração às **14:39**. Oito minutos depois, às **14:47**, o volume de entrada já estava 4 vezes acima do normal e a fila de processamento começou a crescer. Às **15:01**, os dashboards dos clientes pararam de atualizar. Às **15:14**, os alertas passaram a chegar com atraso.

Mas a equipe da ReliOps só **detectou o problema às 15:28** — ou seja, **41 minutos após o início do incidente**.

Aqui há um ponto que vale destacar: a ReliOps é uma empresa que **vende observabilidade para os clientes** — ou seja, ela vende a capacidade de monitorar sistemas em tempo real. Mas o monitoramento do **próprio pipeline interno** não estava bem configurado. Não havia alerta proativo para crescimento anormal da fila. O problema foi percebido de forma reativa, não proativa.

Após a detecção, a equipe agiu rápido: em 3 minutos abriram o war room e em 17 minutos confirmaram o diagnóstico. O incidente foi encerrado às **17:23**, com a fila completamente zerada e o serviço estabilizado.

Passo para [nome da Pessoa 4], que vai explicar por que o incidente aconteceu."

---

### PESSOA 4 — Causa Raiz e 5 Porquês (~3 min)

> *Apresenta a análise de causa raiz usando a técnica dos 5 Porquês.*

---

"Obrigado, [nome da Pessoa 3].

Para identificar a causa raiz do incidente, utilizamos a **técnica dos 5 Porquês** — uma ferramenta de análise que consiste em perguntar 'por quê?' sucessivamente até chegar à origem real do problema, e não apenas ao sintoma.

[ler ou parafrasear a tabela dos 5 Porquês]

Começamos com o sintoma: *por que os clientes receberam alertas atrasados?* Porque o serviço de processamento parou de entregar dados. Por que parou? Porque a fila cresceu mais rápido do que conseguia processar. Por que cresceu sem controle? Porque não havia mecanismo para limitar a entrada em picos. Por que não havia esse mecanismo? Porque o serviço foi configurado só para a carga habitual. E por que não havia planejamento para picos? Porque nenhum incidente similar havia sido documentado anteriormente.

Chegamos à **causa raiz**: o serviço não tinha capacidade de se adaptar a picos de volume, e a ausência de documentação de incidentes anteriores fez com que esse risco nunca tivesse sido endereçado.

Além dos fatores técnicos — como falta de escalonamento automático e ausência de limite na fila — identificamos também **fatores de processo**: nenhum runbook para situações de sobrecarga, e o onboarding do cliente T-2204 não previu o risco do envio em lote de dados históricos. O gatilho do incidente era evitável.

Passo para [nome da Pessoa 5], que vai apresentar como o incidente foi resolvido e quais ações foram definidas."

---

### PESSOA 5 — Mitigação, Trade-offs e Ações (~3 min)

> *Apresenta o que foi feito para resolver, os trade-offs das decisões e as ações preventivas.*

---

"Obrigado, [nome da Pessoa 4].

A equipe aplicou duas ações de emergência para resolver o incidente.

A primeira foi **reduzir temporariamente a velocidade de entrada de novos dados** — o sistema passou a aceitar dados em ritmo menor para dar folga ao serviço de processamento drenar a fila acumulada.

A segunda foi **aumentar manualmente o número de instâncias do serviço de processamento de 3 para 12** — o que acelerou o consumo da fila. Em cerca de 49 minutos depois dessa ação, a fila foi completamente zerada.

É importante deixar claro que essas duas ações são **soluções de contorno**, não soluções definitivas. Elas resolveram a emergência, mas não eliminam a causa raiz. Para chegar a essa conclusão, a equipe avaliou alternativas.

Por exemplo: por que não simplesmente rejeitar os dados do cliente T-2204? Porque rejeição causaria perda de dados — uma violação direta do contrato. Por que não aguardar a fila esvaziar naturalmente? O tempo estimado seria superior a 6 horas — inaceitável dado o SLO.

Para evitar que o incidente se repita, definimos **8 ações** — 5 preventivas e 3 corretivas — todas com responsável e prazo. As de maior prioridade incluem: implementar escalonamento automático até abril, configurar alertas proativos para a fila, e criar um runbook de sobrecarga para que a próxima equipe on-call não precise 'descobrir o que fazer' durante a pressão de um incidente.

Passo para [nome da Pessoa 6], que vai apresentar as lições aprendidas e a parte de governança do documento."

---

### PESSOA 6 — Lições Aprendidas, Governança e Encerramento (~3 min)

> *Apresenta as lições aprendidas, explica a governança documental e encerra.*

---

"Obrigado, [nome da Pessoa 5].

O postmortem levanta algumas lições importantes.

O que **funcionou bem**: após a detecção, a equipe diagnosticou o problema em apenas 17 minutos e agiu de forma coordenada. Os dados não foram perdidos — todos os eventos que ficaram na fila foram processados após a normalização. E a comunicação com os clientes via status page foi ativada ainda durante o incidente.

O que **não funcionou**: a empresa de observabilidade levou 41 minutos para perceber um problema no próprio sistema. A equipe não tinha um runbook, então precisou estruturar a resposta do zero sob pressão. E incidentes informais anteriores nunca foram registrados — por isso, melhorias que poderiam ter prevenido este incidente nunca foram implementadas.

A principal recomendação de processo é adotar a **cultura blameless**: um postmortem não serve para apontar culpados — serve para melhorar o sistema e os processos. O cliente T-2204 agiu de boa-fé. A falha foi da plataforma.

Sobre a **governança do documento**: seguimos o padrão SemVer para versionamento — esta é a versão 1.0.0. O documento tem owner definido, aprovador, data de próxima revisão e classificação da informação. Isso garante rastreabilidade e que o documento não fique desatualizado. O arquivo está versionado no repositório Git — o que chamamos de **Doc as Code**: documentação tratada como código, com histórico de alterações e revisão em equipe.

Para encerrar: o checklist de qualidade da disciplina está completamente preenchido — impacto quantificado, linha do tempo com evidências, causa raiz com 5 Porquês, ações com responsável e prazo, premissas, lacunas e riscos documentados.

Agradecemos a atenção do Professor Arimatéia e abrimos para perguntas."

---

## Dicas para a Apresentação

- **Não leiam o documento na tela** — usem o roteiro como guia e falem para a audiência
- **Transições** — cada pessoa deve chamar o próximo pelo nome para manter o fluxo
- **Digrama** — a Pessoa 2 deve mostrar o diagrama de fluxo do documento ao falar sobre o impacto
- **Tabela dos 5 Porquês** — a Pessoa 4 pode ler a tabela linha a linha ou parafraseá-la
- **Tabela de ações** — a Pessoa 5 não precisa ler todas as 8 ações; destaque as 3 de maior prioridade
- **Tempo** — pratiquem uma vez antes; 3 minutos por pessoa passa rápido

---

## Pontos que o Professor Pode Perguntar

| Pergunta provável | Quem responde | Resposta-chave |
|---|---|---|
| Por que SEV2 e não SEV1? | Pessoa 1 ou 4 | SEV1 é interrupção total; o serviço ficou degradado (lento, com atraso), mas não saiu do ar |
| Por que versão 1.0.0? | Pessoa 6 | Primeira versão aprovada; SemVer garante rastreabilidade — PATCH para correções, MINOR para novas seções, MAJOR para reestruturação |
| O que é blameless? | Pessoa 6 | Cultura de postmortem sem apontar culpados — foco em melhorar o sistema e os processos |
| Por que os horários são estimados? | Qualquer pessoa | O cenário não forneceu horários exatos; todas as estimativas estão sinalizadas no documento e registradas nas lacunas |
| O que é Doc as Code? | Pessoa 6 | Documentação versionada no Git, com histórico de alterações e revisão colaborativa — o mesmo rigor aplicado ao código fonte |
| Qual foi a maior falha? | Pessoa 3 ou 4 | A própria empresa de observabilidade demorou 41 minutos para perceber o problema no próprio sistema — o monitoramento interno não monitorava a si mesmo |
