# Equipe
- Equipe DocumentAI
- Luis Henrique 2527800
- Matheus Marques 2529350
- Isaac Medeiros 2526728
- Joao Arthur 2526593
- Salvatore Luca 2526770

# Runbook Operacional - StreamFlow

## Metadados do Documento
- Documento: Runbook Operacional de Resposta a Incidentes - StreamFlow
- Versão: 1.0.0
- Status: Em revisão
- Responsável (owner): Operações / SRE
- Aprovador: Gestão de Engenharia
- Última atualização: 29-03-2026
- Próxima revisão: 30-03-2026
- Público-alvo: Analista de plantão, SRE/Plataforma, Engenheiros de Software, Suporte e Gestão de Incidentes
- Classificação da informação: Interna

## Premissas, Lacunas e Riscos (preenchimento obrigatório)

### Premissas (o que está sendo assumido para elaborar o documento):
- Existe operação de plantão (on-call) para atendimento inicial de incidentes.
- Há pelo menos uma maneira de monitoramento e logs da aplicação.
- Há pelo menos um canal interno para gestão de incidente e um canal informal para comunicação com suporte.
- Há acesso às ferramentas de monitoramento.
- O serviço está hospedado na AWS.

### Lacunas de informação (dados ausentes que impactam o detalhamento):
- Não há topologia detalhada da arquitetura, dependências de banco, cache, broker e desenho de failover.
- Não há definição explícita de provedor de CDN, contrato de suporte, severidade vigente ou SLOs atuais.
- Não há histórico de incidentes anteriores com MTTA/MTTR reais.
- Não foi informado se existe provedor secundário de CDN, modo degradado ou automação de rollback.
- Ausência prévia de checklist de diagnóstico.
- Inexistência de uma trilha documentada para análise de pós-incidente.

### Riscos identificados (inclua impacto e mitigação sugerida):
- **Risco:** Resolução de incidentes atrasada por escalonamentos informais, como mensagens diretas entre integrantes da equipe.  
  **Impacto:** Aumento do tempo de resposta e de recuperação do incidente, acionamento tardio do time correto, falta de rastreabilidade das decisões e condução inconsistente entre plantões.  
  **Mitigação sugerida:** Definir um fluxo formal de escalonamento com canal único de incidentes, critérios objetivos de severidade (SEV1 a SEV4), responsáveis por nível de atendimento e registro obrigatório de todas as atualizações no canal oficial do incidente.

- **Risco:** Perda de logs e evidências importantes devido ao reinício manual de serviços antes da investigação inicial.  
  **Impacto:** Dificuldade para identificar a causa raiz, comprometimento da análise técnica, aumento da chance de recorrência e possibilidade de adoção de ações corretivas inadequadas.  
  **Mitigação sugerida:** Estabelecer checklist obrigatório de diagnóstico inicial antes de qualquer reinício, incluindo coleta de logs, métricas, horário do erro, estado do serviço e dependências relacionadas. Reinícios manuais só devem ocorrer após captura mínima de evidências ou em situação crítica devidamente registrada.

- **Risco:** Insatisfação de clientes devido à comunicação externa apenas reativa, ocorrendo somente após questionamentos ou abertura de chamados.  
  **Impacto:** Percepção de falta de transparência, aumento de demanda no suporte, desgaste da confiança do cliente e piora da experiência durante incidentes visíveis.  
  **Mitigação sugerida:** Definir gatilhos formais para comunicação externa, com acionamento proativo da equipe de suporte e atualização da status page em incidentes SEV1 e SEV2, especialmente quando houver impacto perceptível ao usuário por período superior ao limite estabelecido no runbook.

## 1. Objetivo do Runbook
- Serviço/sistema coberto:
  - Operação da plataforma StreamFlow com foco nos fluxos críticos de catálogo, processamento de eventos de reprodução e integração com CDN.

- Cenários de aplicação:
  - Pico de erro 5xx no serviço de catálogo em horários de lançamento.
  - Atraso no processamento de eventos de reprodução em horários de pico.
  - Falha intermitente em integração com provedor de CDN.

- Limites de uso deste runbook:
  - Focado em mitigação rápida e recuperação de serviços
  - Não cobre incidentes de segurança, vazamento de dados ou indisponibilidade total de infraestrutura multi-serviço sem análise adicional.
  - Não autoriza mudanças estruturais sem aprovação do responsável técnico/incident manager.

## 2. Equipe e Responsabilidades
| Papel | Responsabilidade | Contato/Canais |
|---|---|---|
| Analista de plantão | Receber alerta, abrir incidente, executar pré-checagens e primeira mitigação segura | Canal interno de incidentes / Slack |
| Engenharia de Eventos/Dados | Apoiar análise de filas, consumidores, backlog e consistência de eventos | Escalonamento técnico de eventos |
| Plataforma/SRE | Atuar em escalabilidade, dependências, deploy, rollback e observabilidade | Slack/Discord |
| Suporte/Atendimento | Consolidar impacto percebido por clientes e apoiar comunicação externa | Canal de suporte / status page / atendimento |


## 3. Critérios de Severidade
| Severidade | Definição | Exemplo de impacto | SLA de resposta |
|---|---|---|---|
| SEV1 | Indisponibilidade crítica ou degradação severa com impacto relevante em evento ao vivo ou jornada principal de muitos usuários | Catálogo indisponível em lançamento de grande audiência; reprodução impactada em massa; falha ampla de entrega via CDN | Imediato |
| SEV2 | Degradação alta com impacto perceptível, mas parcial, mantendo operação degradada | Aumento relevante de 5xx em parte do catálogo; atraso de eventos crescendo; falha intermitente de CDN sem indisponibilidade total | Início do atendimento em até 10 min |
| SEV3 | Incidente moderado, localizado ou intermitente, sem impacto amplo contínuo | Lentidão localizada, backlog controlado, erro intermitente com workaround disponível | Início do atendimento em até 30 min |
| SEV4 | Alerta sem impacto relevante ao usuário ou item preventivo/baixo risco | Oscilação pontual sem impacto confirmado; erro recuperado automaticamente | Tratativa em horário comercial ou até 4h |

## 4. Pré-checagens Operacionais
- Verificar painéis de monitoramento:
  - Validar taxa de erro, latência, saturação, tráfego, backlog e health checks.
  - Confirmar se o alerta é atual, persistente e não um falso positivo.
  - Identificar escopo do impacto: global, regional, por funcionalidade ou por grupo de usuários.

- Validar status de dependências:
  - Banco de dados, cache, fila/broker, autenticação, APIs internas e provedor de CDN.
  - Validar se há degradação em rede, DNS, TLS/certificados ou quota/limite externo.
  - Confirmar status do último deploy e de mudanças recentes em configuração/feature flags.

- Conferir incidentes correlacionados:
  - Verificar incidentes já abertos, mudanças em andamento e manutenções planejadas.
  - Confirmar se suporte já recebeu aumento de chamados relacionados.
  - Consultar status page de dependências externas, quando aplicável.

## 5. Cenários de Incidente (preencher por cenário)

### 5.1 Cenário: Pico de erro 5xx no serviço de catálogo em horário de lançamento
**Sinais e gatilhos:**
- Alerta de aumento de 5xx acima do limiar operacional.
- Queda de sucesso em endpoints do catálogo.
- Aumento de latência durante janela de lançamento.
- Reclamações de usuários sobre catálogo indisponível, vazio ou inconsistente.

**Diagnóstico inicial:**
1. Confirmar o escopo do impacto: todos os usuários, região específica ou apenas endpoints do catálogo.
2. Verificar se houve deploy, alteração de configuração, feature flag ou aumento abrupto de tráfego nos últimos 30 minutos.
3. Analisar métricas de CPU, memória, conexões, pool, banco e cache do serviço de catálogo.
4. Inspecionar logs de erro e classificar o 5xx predominante (timeout, dependência indisponível, erro de aplicação, saturação).
5. Validar health check do serviço e banco associados ao catálogo.
6. Confirmar se há correlação com lançamento específico, campanha ou evento ao vivo.
7. Não reinicie o serviço manualmente de forma imediata; capture os logs antes de qualquer ação destrutiva.

**Ações de mitigação:**
1. Se houver saturação de recurso, escalar horizontalmente o serviço de catálogo e/ou dependências associadas conforme procedimento operacional vigente.
2. Se houver correlação com deploy/configuração recente, interromper avanço da mudança e iniciar rollback para a última versão/configuração estável.
3. Se o problema estiver concentrado em funcionalidade recém-lançada, desabilitar feature flag correspondente, quando existir.
4. Reiniciar instâncias apenas se a causa provável indicar travamento, vazamento de recurso ou recuperação já validada historicamente.

**Critério de escalonamento:**
- Quando escalar:
  - Imediatamente em SEV1.
  - Se o erro 5xx persistir por mais de 5 minutos após a primeira mitigação.
  - Se houver impacto em evento ao vivo, lançamento relevante ou crescimento contínuo da taxa de erro.
  - Se houver suspeita de falha em dependência crítica ou necessidade de rollback.
- Para quem escalar:
  - Plataforma/SRE

**Comunicação:**
- Canal interno:
  - Abrir incidente no canal oficial de incidentes com timestamp, severidade, escopo, hipótese inicial e ações em andamento.
- Canal externo (quando aplicável):
  - Acionar suporte/status page para SEV1 ou SEV2 com impacto ao usuário por mais de 15 minutos.
  - Atualizar comunicação a cada 15 minutos enquanto durar o incidente crítico.
  - Acionar a equipe de suporte para enviar comunicação ativa na status page. Não aguarde o cliente reportar o erro primeiro.

---

### 5.2 Cenário: Atraso no processamento de eventos de reprodução em horário de pico
**Sinais e gatilhos:**
- Aumento de lag/backlog em fila ou stream de eventos.
- Queda na taxa de consumo/processamento.
- Atraso na disponibilidade de métricas/telemetria de reprodução.
- Alertas de timeout ou falha em workers/consumidores.

**Diagnóstico inicial:**
1. Confirmar se o atraso é crescente, estável ou já em recuperação automática.
2. Verificar backlog atual, throughput de entrada/saída, taxa de erro dos consumidores e tempo médio de processamento.
3. Validar estado do broker/fila/stream, consumidores, workers e storage de destino.
4. Identificar mudanças recentes em jobs, deploys, schemas, feature flags ou cargas extraordinárias.
5. Confirmar impacto de negócio: apenas telemetria atrasada ou reflexo em experiência do usuário e operação.

**Ações de mitigação:**
1. Escalar consumidores/workers responsáveis, respeitando limites seguros do broker e do storage.
2. Pausar temporariamente cargas não essenciais ou processamento secundário para priorizar o fluxo crítico.
3. Se houver falha em versão recente de consumidor/job, executar rollback para a versão estável anterior.
4. Reprocessar backlog apenas após validar idempotência, ordem e risco de duplicidade.
5. Se houver gargalo em dependência externa/interna, acionar time responsável e reduzir pressão no pipeline até estabilização.

**Critério de escalonamento:**
- Quando escalar:
  - Se o backlog continuar crescendo por 10 minutos após mitigação inicial.
  - Se houver risco de perda, duplicidade ou corrupção de eventos.
  - Se o atraso impactar indicadores operacionais críticos ou jornada do usuário.
  - Se houver dependência de intervenção em broker, storage ou schema.
- Para quem escalar:
  - Plataforma/SRE
  - Equipe de suporte
  - Gestão de Engenharia em caso de SEV1

**Comunicação:**
- Canal interno:
  - Registrar backlog, tempo de atraso, hipótese principal e mitigação executada.
- Canal externo (quando aplicável):
  - Informar suporte caso o atraso afete funcionalidades percebidas pelo usuário ou relatórios críticos internos.
  - Comunicação externa apenas quando houver impacto visível ao cliente final.

---

### 5.3 Cenário: Falha intermitente na integração com provedor de CDN
**Sinais e gatilhos:**
- Erros intermitentes de timeout, 4xx/5xx ou falha de entrega em integrações com a CDN.
- Oscilação em disponibilidade de ativos/segmentos.
- Aumento de reclamações de buffering, falha de carregamento ou degradação regional.
- Divergência entre métricas internas e comportamento reportado por suporte.

**Diagnóstico inicial:**
1. Confirmar se a falha é global, regional, por tipo de conteúdo ou por operação específica da integração.
2. Verificar logs de chamadas para a CDN, códigos de erro, latência e taxa de timeout.
3. Consultar status do provedor e incidentes reportados externamente.
4. Validar credenciais, certificados, DNS, conectividade e limites/quota da integração.
5. Identificar se houve mudança recente de configuração de rota, cache, invalidação ou política de entrega.
6. Verificar se existe provedor secundário, fallback de rota ou modo de contingência documentado.

**Ações de mitigação:**
1. Se houver capacidade prevista, acionar fallback para rota/provedor alternativo ou perfil de entrega secundário.
2. Reduzir operações não essenciais de integração com a CDN (ex.: invalidações em massa, rotinas administrativas) até estabilização.
3. Se a falha estiver associada a mudança recente de configuração, executar rollback controlado da configuração afetada.
4. Manter serviços internos estáveis e evitar ações que ampliem a pressão sobre a dependência externa.
5. Acionar fornecedor/CDN com evidências mínimas: horário, região, códigos de erro, volume afetado e impacto percebido.

**Critério de escalonamento:**
- Quando escalar:
  - Se houver impacto em reprodução/entrega de conteúdo para usuários.
  - Se a falha persistir por mais de 10 minutos sem estabilização.
  - Se não houver fallback interno disponível.
  - Se for necessário envolver o fornecedor externo ou gestão.
- Para quem escalar:
  - Plataforma/SRE
  - Responsável por Integrações/CDN
  - Suporte
  - Gestão de Engenharia em caso de SEV1

**Comunicação:**
- Canal interno:
  - Reunião imediada para tratar do incidente com atualização de escopo, regiões afetadas, mitigação e resposta do fornecedor.
- Canal externo (quando aplicável):
  - Suporte e status page para incidentes com impacto perceptível ao usuário.
  - Em SEV1, publicar atualização inicial em até 15 minutos após confirmação do impacto.

## 6. Rollback e Recuperação
- Situações que exigem rollback:
  - Evidência de correlação entre incidente e deploy/configuração/feature flag recente.
  - Piora do incidente após mudança aplicada como mitigação.
  - Regressão funcional confirmada em serviço crítico.
  - Incapacidade de estabilização por mitigação de baixo risco em tempo aceitável.

- Procedimento padrão:
  1. Confirmar a última versão/configuração conhecida como estável.
  2. Registrar no canal do incidente a decisão de rollback, motivo, responsável e escopo.
  3. Interromper novas mudanças no serviço afetado até estabilização.
  4. Executar rollback controlado conforme pipeline/procedimento da equipe responsável.
  5. Monitorar indicadores críticos por pelo menos 15 minutos após o rollback.
  6. Manter o incidente aberto até confirmação de estabilidade e comunicação final.

- Validação pós-recuperação:
  - Taxa de erro normalizada.
  - Latência dentro do padrão operacional.
  - Backlog estabilizado ou em redução contínua.
  - Health checks aprovados.
  - Ausência de novos alertas correlatos por janela mínima de observação.
  - Suporte validou redução/cessação de sintomas reportados pelos clientes.

## 7. Pós-Incidente
- Registro obrigatório:
  - Identificação do incidente
  - Data/hora de detecção
  - Data/hora de início do atendimento
  - Severidade atribuída
  - Serviços afetados
  - Impacto ao usuário/negócio
  - Timeline resumida das ações
  - Causa provável/causa raiz
  - Mitigação aplicada
  - Necessidade de rollback
  - Responsáveis envolvidos
  - Lições aprendidas
  - Ações preventivas e respectivos donos

- Template de análise de causa raiz:
  - O que aconteceu?
  - Quando começou?
  - Como foi detectado?
  - Qual foi o impacto?
  - Qual a causa raiz confirmada?
  - Quais sinais poderiam ter antecipado o incidente?
  - O que funcionou bem na resposta?
  - O que atrasou a recuperação?
  - Quais ações corretivas/preventivas serão executadas?
  - Quem é o responsável e qual o prazo?

- Ações preventivas:
  - Revisar e complementar este runbook após cada incidente relevante.
  - Adicionar dashboards, queries e links reais nos anexos.
  - Formalizar critérios de severidade e comunicação com suporte/status page.
  - Criar checklists específicos por serviço crítico.
  - Avaliar automações seguras para mitigação recorrente e rollback.

## 8. Indicadores Operacionais
- MTTA:
  - Meta inicial: até 5 min para SEV1, 10 min para SEV2, 30 min para SEV3.
- MTTR:
  - Meta inicial: até 30 min para mitigação de SEV1, 60 min para SEV2 e 120 min para SEV3.
- Taxa de recorrência:
  - Meta inicial: no máximo 1 recorrência relevante do mesmo incidente em 30 dias; acima disso, RCA obrigatória com ação preventiva priorizada.

## 9. Riscos e Limitações
- Dependências críticas:
  - Serviço de catálogo
  - Pipeline de processamento de eventos
  - Broker/fila/stream
  - Banco/cache associados
  - Provedor de CDN
  - Ferramentas de monitoramento e comunicação operacional

- Pontos sem cobertura:
  - Incidentes de segurança e fraude
  - Disaster recovery regional/multi-região
  - Falhas amplas de identidade/autenticação fora do escopo descrito
  - Procedimentos detalhados de fornecedor externo
  - Scripts e comandos reais ainda não anexados

- Melhorias futuras:
  - Incluir links reais para dashboards, logs e pipelines.
  - Adicionar fluxograma Mermaid por cenário.
  - Incluir matriz RACI com nomes reais.
  - Refinar SLAs com base em histórico operacional.
  - Criar runbooks complementares por serviço e por dependência crítica.

## Anexos e Referências
- Painéis de monitoramento e alertas:
  - [Inserir links reais dos dashboards de catálogo, eventos e CDN]
- Comandos/scripts operacionais de apoio:
  - [Inserir comandos seguros de consulta, restart, rollback e validação]
- Canal/ticket de incidentes relacionados:
  - [Inserir canal oficial e sistema de tickets/incidentes]
- Links de PRs/issues relacionados:
  - [Inserir links do fork, PRs de revisão e issues de melhoria]

## Checklist de Qualidade (pré-entrega)
- [x] Critérios de acionamento e severidade definidos.
- [x] Diagnóstico inicial e mitigação passo a passo claros.
- [x] Escalonamento e comunicação descritos.
- [x] Rollback e validação pós-recuperação documentados.
- [x] Premissas, lacunas e riscos preenchidos.