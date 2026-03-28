# INC-001: Sobrecarga do Serviço de Processamento por Volume Inesperado de Eventos

- **Status:** Concluído
- **Severidade:** SEV2
- **Período do incidente:** 2026-03-10 14:47 até 2026-03-10 17:23
- **Autores:** Grupo ReliOps — Pós-graduação UNIFOR
- **Times envolvidos:** SRE, Engenharia de Plataforma, Suporte ao Cliente

## Metadados de Governança

- Documento: postmortem-inc-001.md
- Versão: 1.0.0
- Responsável (owner): SRE Lead — Grupo ReliOps
- Aprovador: Prof. Arimatéia Júnior
- Última atualização: 2026-03-28
- Próxima revisão: 2026-06-28
- Classificação da informação: Interna

> **Por que versão 1.0.0?** Esta é a primeira versão aprovada e publicada deste postmortem. Correções pontuais futuras resultam em 1.0.1 (PATCH). Adição de nova seção de aprendizado resulta em 1.1.0 (MINOR). Reestruturação completa resulta em 2.0.0 (MAJOR). Esse padrão (SemVer) garante rastreabilidade do histórico documental.

### Histórico de Versões

| Versão | Data | Autor | Alteração |
|---|---|---|---|
| 1.0.0 | 2026-03-28 | Grupo ReliOps — UNIFOR | Versão inicial aprovada e publicada — cobre todo o ciclo do INC-001 |

---

## Premissas, Lacunas e Riscos (preenchimento obrigatório)

**Premissas** — o que está sendo assumido para elaborar o documento:

1. As informações descritas em `src/incidente-resumo.md` representam fielmente o que ocorreu durante o incidente.
2. A duração estimada de 2h 36min é baseada na sequência lógica dos eventos descritos no cenário, já que horários exatos não foram fornecidos.
3. O SLO de 99,9% de disponibilidade mensal é o contrato de referência para calcular o impacto ao cliente.
4. A solução emergencial aplicada foi eficaz e o serviço retornou ao estado normal após a mitigação.
5. O cliente T-2204 agiu de boa-fé ao ativar a integração — não havia intenção de causar sobrecarga na plataforma.

**Lacunas de informação** — dados ausentes que impactam o detalhamento:

1. Horários exatos de cada evento do incidente não estão disponíveis — todos os horários da linha do tempo são estimados com base na ordem dos acontecimentos.
2. O número exato de clientes afetados e tickets de suporte abertos não foi informado no cenário — utilizamos estimativa de ~380 tenants.
3. Dados técnicos de desempenho (uso de CPU, memória, tamanho da fila em tempo real) não foram disponibilizados no cenário fornecido.
4. Não há confirmação se outros clientes também enviaram volumes anormais simultaneamente ao cliente T-2204.

**Riscos identificados** — inclua impacto e mitigação sugerida:

| Risco | Impacto | Mitigação |
|---|---|---|
| A causa raiz identificada é uma hipótese — não há logs confirmando tecnicamente | As ações corretivas podem não resolver o problema real | Validar a hipótese após implementar as melhorias e revisar o documento se necessário (versão 1.0.1) |
| Estimativa de clientes afetados pode estar abaixo do real | Comunicação incompleta com os clientes impactados | Cruzar com dados de tickets de suporte ao executar as ações corretivas |
| Ações preventivas podem ficar sem implementação por falta de acompanhamento | O incidente pode se repetir sem que as melhorias tenham sido aplicadas | Revisar o status de todas as ações na reunião de equipe prevista para 2026-06-28 |

---

## 1. Resumo Executivo

**O que aconteceu:**

No dia 10/03/2026, o cliente T-2204 ativou uma nova integração com a plataforma ReliOps e, ao fazer isso, disparou o envio de um grande volume de dados históricos em um curto período de tempo. O volume enviado foi aproximadamente 4 vezes maior do que o serviço de processamento estava configurado para suportar.

Com a entrada repentina de um volume muito acima do normal, a fila interna de eventos a serem processados cresceu rapidamente. O serviço de processamento não conseguiu acompanhar o ritmo de entrada, o que impediu que dados atualizados chegassem aos dashboards e ao sistema de alertas dos clientes. Cerca de 380 clientes ficaram com dashboards desatualizados e receberam alertas com até 34 minutos de atraso durante 2h 36min.

**Impacto principal:**

Degradação severa dos serviços de alertas e dashboards para ~380 clientes durante 2h 36min. O SLO contratado de 99,9% de disponibilidade mensal foi violado — o budget de erro de ~44 minutos/mês foi consumido em ~356% por este único incidente.

**Situação atual:**

Serviço restaurado. A equipe de SRE aplicou mitigação emergencial (aumento de capacidade manual + redução temporária de entrada de dados) e o sistema foi normalizado sem perda de dados. Este postmortem documenta formalmente o incidente e define as ações para evitar recorrência.

---

## 2. Impacto no Negócio e nos Usuários

### Diagrama de Fluxo da Plataforma ReliOps

**Operação normal:**

```
┌─────────────────┐     ┌──────────────────┐     ┌────────────────────────┐     ┌───────────┐
│    Clientes      │────▶│  API de Ingestão  │────▶│  Serviço de            │────▶│  Banco de │
│  (~380 tenants)  │     │                  │     │  Processamento         │     │  Dados    │
└─────────────────┘     └──────────────────┘     │  (3 instâncias)        │     └─────┬─────┘
                                                  └────────────────────────┘           │
                                                                                        ▼
                                                                              ┌──────────────────┐
                                                                              │  Dashboards e    │
                                                                              │  Alertas         │
                                                                              │  (tempo real)    │
                                                                              └──────────────────┘
```

**Durante o incidente (14:47 – 17:23):**

```
┌─────────────────┐     ┌──────────────────┐     ┌────────────────────────┐
│  Clientes        │     │  API de Ingestão  │     │ ⚠ Fila acumulando      │
│  (~380 tenants)  │────▶│  (sem controle   │────▶│    sem limite          │    ✗ Banco sem atualização
│                  │     │   de volume)     │     │    (backlog crescente) │
│ + T-2204 (4x    │     └──────────────────┘     └────────────────────────┘
│   volume normal) │                                          │
└─────────────────┘                                          ▼
                                                  ┌──────────────────────────────┐
                                                  │  Dashboards congelados        │
                                                  │  Alertas com até 34min atraso │
                                                  └──────────────────────────────┘
```

**Serviços afetados:**

- Serviço de ingestão de eventos — operando, mas aceitando muito mais dados do que o sistema conseguia processar
- Serviço de processamento de eventos — sobrecarregado, acumulando fila de forma crescente e descontrolada
- Dashboards dos clientes — congelados, sem atualização de dados em tempo real
- Sistema de alertas — entregando notificações com atraso de até 34 minutos

**Janela de indisponibilidade/degradação:**

2h 36min de degradação severa (das 14:47 às 17:23 do dia 10/03/2026).

**Impacto percebido pelo cliente:**

- O SLO contratado garante no máximo ~44 minutos de problemas por mês (equivalente a 99,9% de disponibilidade). Este incidente sozinho causou ~156 minutos de degradação — mais de 3 vezes o limite mensal permitido.
- Cerca de 380 clientes ficaram sem visibilidade em tempo real sobre as suas próprias aplicações durante a janela do incidente. Alertas atrasados significam que problemas nas aplicações dos clientes poderiam ter sido identificados com até 34 minutos de atraso — comprometendo diretamente o valor contratado.
- Ao menos 3 clientes enterprise abriram tickets de suporte durante o incidente.

---

## 3. Linha do Tempo do Incidente

| Horário | Evento | Evidência/Fonte | Responsável |
|---|---|---|---|
| 14:39 | Cliente T-2204 ativa nova integração e inicia envio de volume alto de dados históricos | Log do sistema de ingestão (estimado) | Cliente T-2204 (externo) |
| 14:47 | Volume de dados recebidos chega a ~4x o normal — fila de processamento começa a crescer | Alerta de volume interno (estimado) | Sistema |
| 14:55 | Fila de processamento cresce de forma contínua — serviço não consegue processar no ritmo de entrada | Hipótese baseada nos sintomas do cenário | Serviço de processamento |
| 15:01 | Dashboards dos clientes param de atualizar — dados ficam congelados | Sintoma descrito em src/incidente-resumo.md | Plataforma |
| 15:14 | Alertas passam a ser entregues com atraso superior a 5 minutos | Sintoma descrito em src/incidente-resumo.md | Sistema de alertas |
| 15:28 | Equipe de SRE identifica o problema via alerta interno do próprio sistema de monitoramento | Alerta interno da ReliOps | SRE on-call |
| 15:31 | War room iniciado — responsável pelo incidente designado | Registro interno | SRE Lead |
| 15:45 | Diagnóstico confirmado: sobrecarga do serviço de processamento causada pelo volume excessivo do cliente T-2204 | Análise de métricas internas | Engenharia de Plataforma |
| 15:48 | Status page pública atualizada informando os clientes sobre a degradação | Comunicação pública | Suporte ao Cliente |
| 16:03 | Redução temporária da velocidade de entrada de novos dados aplicada para aliviar a pressão | Ação de mitigação | SRE |
| 16:15 | Capacidade do serviço de processamento aumentada manualmente — número de instâncias ampliado de 3 para 12 | Ação de mitigação | SRE |
| 16:34 | Fila começa a reduzir — serviço de processamento supera o ritmo de entrada | Observação operacional | SRE |
| 17:05 | Dashboards voltam a atualizar — alertas entregues em tempo real sem atraso | Observação operacional | SRE Lead |
| 17:23 | Fila zerada — incidente encerrado e estabilização confirmada | Critério de estabilização atingido | SRE Lead |
| 17:40 | Velocidade de entrada de dados normalizada gradualmente | Normalização controlada | SRE |

---

## 4. Detecção e Resposta Inicial

**Como o incidente foi detectado:**

O sistema de monitoramento interno da ReliOps disparou um alerta com ~41 minutos de atraso em relação ao início do problema (14:47 → 15:28). Este dado é especialmente relevante: a ReliOps fornece observabilidade para os seus clientes, mas o monitoramento do próprio pipeline interno de processamento não estava adequadamente configurado — não havia alertas proativos para crescimento anormal da fila de eventos. O incidente foi detectado de forma reativa, não proativa.

**Tempo até início da resposta:**

- Tempo até detecção: ~41 minutos após o início do problema
- Tempo até início da resposta: ~3 minutos após a detecção (15:28 → 15:31)
- Tempo até diagnóstico confirmado: ~17 minutos após a detecção (15:28 → 15:45)
- Tempo total até estabilização: 2h 36min

**Medidas imediatas adotadas:**

1. Redução temporária da velocidade de entrada de novos dados — deu folga ao serviço de processamento para drenar a fila acumulada
2. Aumento manual do número de instâncias do serviço de processamento (de 3 para 12) — acelerou o consumo da fila
3. Acompanhamento manual da fila a cada 5 minutos até drenagem completa
4. Atualização da status page pública informando os clientes sobre a degradação em curso

**Log de comunicação com stakeholders:**

| Horário | Canal | Destinatário | Conteúdo comunicado | Responsável |
|---|---|---|---|---|
| 15:48 | Status page pública | Todos os clientes da plataforma | Comunicado de degradação em andamento — dashboards e alertas impactados | Suporte ao Cliente |
| 16:10 | E-mail direto | 3 clientes enterprise que abriram ticket | Atualização personalizada sobre o incidente + estimativa de resolução | Suporte ao Cliente |
| 17:30 | Status page pública | Todos os clientes da plataforma | Comunicado de resolução — serviço normalizado, sem perda de dados | Suporte ao Cliente |
| 17:45 | E-mail direto | 3 clientes enterprise | Confirmação da resolução + resumo do impacto + próximos passos | Suporte ao Cliente |

---

## 5. Causa Raiz e Fatores Contribuintes

**Causa raiz:**

O serviço de processamento não estava preparado para absorver picos de volume muito acima do padrão habitual de uso. Não havia mecanismo automático para ampliar a capacidade do serviço quando a demanda crescesse de forma inesperada. Combinado a isso, o processo de onboarding do cliente T-2204 não previu e não validou o cenário de envio retroativo de dados históricos em lote — tornando o gatilho do incidente evitável.

**Análise de causa raiz — Técnica dos 5 Porquês:**

| # | Pergunta | Resposta |
|---|---|---|
| 1 | Por que os clientes receberam alertas atrasados e dashboards desatualizados? | Porque o serviço de processamento parou de entregar dados atualizados em tempo real |
| 2 | Por que o serviço de processamento parou de funcionar no ritmo normal? | Porque a fila de eventos cresceu muito mais rápido do que o serviço conseguia processar |
| 3 | Por que a fila cresceu sem controle? | Porque o sistema não tinha nenhum mecanismo para limitar ou absorver automaticamente picos de volume na entrada |
| 4 | Por que não havia esse mecanismo? | Porque o serviço foi configurado para a carga média habitual, sem considerar cenários de pico intenso ou envio em lote |
| 5 | Por que não havia planejamento para picos? | Porque não existia processo formal de revisão de capacidade e nenhum incidente similar havia sido documentado anteriormente para embasar essa decisão |

**Fatores técnicos contribuintes:**

- Capacidade do serviço de processamento configurada apenas para o volume médio histórico, sem margem para picos
- Ausência de mecanismo automático de escalonamento (a capacidade precisou ser ampliada manualmente pela equipe)
- Sem limite definido para o tamanho máximo da fila de eventos
- Sem alerta proativo configurado para detectar crescimento anormal da fila antes que impactasse os clientes

**Fatores processuais contribuintes:**

- Nenhum postmortem documentado de incidentes anteriores similares — aprendizados anteriores se perderam
- Ausência de manual operacional (runbook) para situações de sobrecarga do serviço de processamento
- Processo de onboarding de novos clientes sem etapa de validação do padrão e volume de envio de dados
- Ações informais tomadas em ocorrências passadas foram distribuídas sem responsável e prazo — nunca foram implementadas

---

## 6. Mitigação e Recuperação

**Ações que restauraram o serviço:**

**Ação 1 — Redução da velocidade de entrada de dados (16:03):**
O sistema passou a aceitar novos dados em um ritmo menor, aliviando a pressão sobre o serviço de processamento e permitindo que a fila acumulada fosse gradualmente drenada. Esta é uma solução de contorno — eficaz em emergência, mas que impacta negativamente os clientes que tentam enviar dados durante o período de restrição.

**Ação 2 — Aumento manual da capacidade de processamento (16:15):**
A equipe ampliou o número de instâncias do serviço de processamento de 3 para 12, acelerando o consumo da fila. Em aproximadamente 49 minutos após a ação, a fila foi completamente zerada. Esta também é uma solução de contorno — requer intervenção manual e não acontece de forma automática quando necessário.

**Ação 3 — Monitoramento manual da fila:**
Acompanhamento a cada 5 minutos até confirmação da estabilização completa.

**Alternativas consideradas e trade-offs:**

| Decisão tomada | Alternativa descartada | Por que foi descartada |
|---|---|---|
| Throttling (reduzir entrada de dados) | Rejeitar completamente os dados do cliente T-2204 | Rejeição causaria perda definitiva de dados do cliente — violação direta do contrato de serviço |
| Aumentar instâncias manualmente (3 → 12) | Aguardar a fila esvaziar naturalmente | O tempo estimado de esvaziamento natural era superior a 6 horas — inaceitável dado o SLO |
| Normalização gradual da entrada ao encerrar | Restaurar a velocidade normal imediatamente | Restauração imediata arriscava gerar um segundo pico de processamento, prolongando o incidente |
| Escalonamento para 12 instâncias | Escalonamento mais agressivo (20+ instâncias) | O gargalo era a fila, não capacidade ilimitada — 12 instâncias eram suficientes para superar o ritmo de entrada |

**Estratégia de rollback/contorno:**

As duas ações aplicadas são workarounds — resolvem a emergência, mas não eliminam a causa raiz. A solução definitiva é implementar escalonamento automático (a capacidade do serviço deve crescer sozinha quando a demanda aumentar) e configurar um mecanismo que limite a entrada de dados quando a fila ultrapassar um volume seguro. Estas soluções estão previstas na Seção 7 (Ações 1 e 5). Prazo para implementação: 2026-04-30 e 2026-05-30, respectivamente.

**Critério de estabilização:**

O incidente foi declarado encerrado às 17:23 quando os seguintes critérios foram confirmados:
- Fila de eventos zerada (nenhum evento aguardando processamento)
- Dashboards atualizando normalmente com atraso menor que 15 segundos
- Alertas sendo entregues em tempo real, dentro do SLO
- Serviço de processamento estável com volume de entrada normalizado

---

## 7. Ações Corretivas e Preventivas

| Ação | Tipo (Corretiva/Preventiva) | Prioridade | Responsável | Prazo | Status |
|---|---|---|---|---|---|
| Implementar escalonamento automático do serviço de processamento — aumentar capacidade automaticamente em momentos de pico, sem necessidade de intervenção manual | Preventiva | Alta | Engenharia de Plataforma | 2026-04-30 | Pendente |
| Configurar alerta proativo para crescimento anormal da fila de eventos — disparar antes que o impacto chegue aos clientes | Corretiva | Alta | SRE | 2026-04-10 | Pendente |
| Ajustar a capacidade padrão do serviço de processamento para suportar picos de até 3x o volume habitual sem intervenção | Corretiva | Alta | Engenharia de Plataforma | 2026-04-10 | Pendente |
| Criar manual operacional (runbook) para situações de sobrecarga do serviço de processamento — com passos de diagnóstico, escalonamento e critério de estabilização | Corretiva | Média | SRE | 2026-04-15 | Pendente |
| Criar mecanismo que limite automaticamente a entrada de dados quando a fila ultrapassar um volume máximo seguro | Preventiva | Média | Engenharia de Plataforma | 2026-05-30 | Pendente |
| Incluir no processo de onboarding de novos clientes uma etapa de validação do volume e padrão de envio de dados — especialmente para integrações com envio em lote | Preventiva | Média | Produto + SRE | 2026-05-15 | Pendente |
| Estabelecer revisão trimestral da capacidade do pipeline de ingestão e processamento | Preventiva | Baixa | Engenharia de Plataforma | 2026-06-30 | Pendente |
| Criar template institucional de postmortem versionado no repositório (Doc as Code) — garantir que incidentes futuros sejam documentados formalmente | Corretiva | Alta | SRE | 2026-03-28 | **Concluído** |

---

## 8. Lições Aprendidas

**O que funcionou bem:**

- Após a detecção, o diagnóstico foi feito em apenas 17 minutos — demonstrando boa capacidade analítica da equipe sob pressão
- O escalonamento manual da capacidade de processamento funcionou como válvula de emergência eficaz
- A colaboração entre SRE e Engenharia de Plataforma foi fluida e coordenada durante toda a resposta
- Os dados não foram perdidos — os eventos ficaram aguardando na fila e foram todos processados após a normalização
- A comunicação com os clientes via status page foi ativada ainda durante o incidente

**O que não funcionou:**

- A própria empresa de observabilidade demorou 41 minutos para detectar um problema no seu próprio sistema interno — o monitoramento da plataforma não monitorava a si mesma adequadamente
- A ausência de um manual operacional (runbook) fez com que a equipe precisasse estruturar a resposta "do zero" em um momento de alta pressão
- Incidentes informais anteriores não foram documentados — por isso, melhorias que poderiam ter evitado este incidente nunca foram implementadas
- O processo de onboarding do cliente T-2204 não identificou o risco do envio retroativo de dados em volume alto — o gatilho do incidente era evitável

**Melhorias de processo recomendadas:**

1. Adotar **Doc as Code para postmortems**: documentos versionados no repositório, com revisão em equipe antes de publicar — garante que o aprendizado de cada incidente fique registrado e acessível para toda a organização.
2. Formalizar a **cultura blameless**: o foco de todo postmortem deve ser melhorar o sistema e os processos, não apontar culpados. A linguagem deve ser neutra e orientada ao aprendizado. O cliente T-2204 agiu de boa-fé; a falha foi da plataforma, não do cliente.
3. Definir **SLO interno para tempo de detecção**: uma empresa de observabilidade não pode demorar 41 minutos para perceber um problema no próprio sistema. Proposta: tempo de detecção de incidentes SEV2 internos < 5 minutos.
4. Criar **rotina mensal de revisão das ações preventivas abertas** — evitar que ações definidas em postmortems fiquem esquecidas sem implementação.

---

## 9. Acompanhamento

- **Itens pendentes:** 7 ações abertas (ver Seção 7 — todas com responsável e prazo definidos)
- **Próxima revisão:** 2026-06-28
- **Critério de encerramento do postmortem:** todas as ações da Seção 7 com status "Concluído" e verificadas pelo Owner do documento, com evidência de implementação (código publicado, documentação disponível ou processo formalizado e adotado)

---

## Anexos e Referências

- Evidências (logs, métricas, gráficos): Logs internos de volume e fila — não disponíveis no cenário desta simulação acadêmica; horários e volumes marcados como "(estimado)" ao longo do documento
- Canal/ticket do incidente: INC-001 — canal interno de guerra (war room) ativado em 15:31 do dia 10/03/2026
- Runbooks, RFCs e ADRs relacionados: Runbook para sobrecarga do serviço de processamento — a ser criado (Ação 4, prazo 2026-04-15)
- Links de PRs/issues das ações corretivas: A definir conforme implementação das ações da Seção 7

---

## Checklist de Qualidade (pré-entrega)

- [x] Impacto e janela do incidente quantificados.
- [x] Linha do tempo completa com evidências.
- [x] Causa raiz e fatores contribuintes descritos.
- [x] Ações corretivas/preventivas com dono e prazo.
- [x] Premissas, lacunas, riscos e acompanhamento preenchidos.
