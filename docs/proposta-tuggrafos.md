# TugGrafos — Plataforma de Conhecimento Técnico para Chefes de Máquinas

> **Proposta documental ampla**
> Documento de visão, arquitetura e implementação para um copiloto de
> engenharia naval/industrial que transforma documentos técnicos em
> **grafos de conhecimento navegáveis**, operados por voz e assistidos por
> agentes de IA.

| Campo | Valor |
|---|---|
| Codinome do produto | **TugGrafos** (Tug = rebocador/praça de máquinas · Grafos = grafo de conhecimento) |
| Agente IA "Chefe de Máquinas" | **KRATOS** (Knowledge · Reasoning · Automation · Technical · Ops · Safety) |
| Público-alvo | Chefes de Máquinas, oficiais de máquinas, equipes de manutenção e PMS |
| Versão do documento | 1.0 |
| Data | 2026-06-05 |
| Status | Proposta para aprovação |

---

## 1. Sumário executivo

O **Chefe de Máquinas** é, hoje, um gestor de informação tanto quanto um
gestor de equipamentos. Manuais de fabricante, planos de manutenção (PMS),
boletins técnicos, certificados de classe, relatórios de avaria, listas de
sobressalentes e registros de horas de funcionamento vivem espalhados em
PDFs digitalizados, pastas físicas e e-mails. Encontrar **a peça certa de
informação no momento certo** — durante uma avaria, uma inspeção de PSC
(Port State Control) ou um turno noturno — é lento e propenso a erro.

O **TugGrafos** resolve isso com três movimentos:

1. **Ingestão** — você alimenta o app com documentos (fotos de manuais,
   PDFs digitalizados, planilhas). Um pipeline de **OCR assistido** lê e
   limpa o texto.
2. **Organização** — o agente **KRATOS** estrutura o conteúdo em um
   **grafo de conhecimento**: equipamentos, peças, procedimentos, sintomas,
   causas e ações ligados entre si. O **Recall API** dá memória de longo
   prazo ao agente.
3. **Consulta** — você pergunta por **voz** ou texto ("qual o torque do
   parafuso da cabeça do cilindro nº 3 do MCP?") e recebe a resposta
   **citando a página e o documento de origem**. O **NotebookLM** gera
   resumos e briefings; o **Grok AI** atua como motor conversacional
   alternativo; a interface é publicada via **Netlify**.

O resultado é um **copiloto de praça de máquinas** que reduz tempo de
diagnóstico, melhora a rastreabilidade para auditorias e preserva o
conhecimento tácito da tripulação.

---

## 2. Problema e oportunidade

### 2.1 Dores atuais
- **Documentação fragmentada**: manuais de OEM em PDF escaneado, sem busca
  textual; revisões e errata desconexas.
- **Conhecimento tácito volátil**: a saída de um chefe experiente leva
  embora "macetes" não documentados.
- **Pressão de tempo**: avarias e inspeções exigem resposta em minutos.
- **Rastreabilidade**: auditorias (ISM, classe, PSC) exigem mostrar
  *de onde* veio cada decisão/procedimento.
- **Conectividade limitada a bordo**: nem sempre há banda para nuvem.

### 2.2 Oportunidade
Combinar **OCR + grafos de conhecimento + RAG (geração aumentada por
recuperação) + voz** num produto enxuto, com **citação de fonte
obrigatória** (anti-alucinação) e modo **offline-first** para uso a bordo.

---

## 3. Visão do produto

> "Alimente o TugGrafos com seus documentos. O KRATOS organiza tudo num
> grafo. Pergunte por voz — receba a resposta com a fonte."

Princípios de design:
- **Fonte sempre citada** — nenhuma resposta sem referência ao documento e
  página. Se não há fonte, KRATOS diz "não encontrado".
- **Offline-first** — núcleo funciona a bordo sem internet; sincroniza em
  porto.
- **Mãos-livres** — voz é cidadã de primeira classe (luvas, óleo, ruído).
- **Segurança operacional** — KRATOS *informa e recomenda*, nunca *aciona*
  equipamento. Decisão é humana.
- **Privacidade** — documentos do armador não treinam modelos de terceiros.

---

## 4. Personas e casos de uso

| Persona | Necessidade | Caso de uso TugGrafos |
|---|---|---|
| Chefe de Máquinas | Diagnóstico rápido de avaria | "KRATOS, o MCP está com alta temperatura de gases no cilindro 4. Causas prováveis?" → grafo lista causas, procedimentos e peças. |
| 1º Oficial de Máquinas | Preparar manutenção planejada | Briefing automático (NotebookLM) do procedimento de revisão das 5.000 h. |
| Eletricista / Mecânico | Achar especificação | Por voz: torque, folga, número de peça, óleo recomendado — com página. |
| Superintendente (terra) | Auditoria / KPIs | Painel web (Netlify) com histórico de consultas e cobertura documental. |
| Inspetor PSC / Classe | Evidência rastreável | Cada resposta exporta a citação da fonte original. |

---

## 5. Arquitetura de solução

### 5.1 Visão em camadas

```
┌──────────────────────────────────────────────────────────────────┐
│  EXPERIÊNCIA (Frontend — publicado na Netlify)                     │
│  • App web/PWA  • Assistente de Voz (STT/TTS)  • Chat KRATOS        │
│  • Visualizador de Grafo  • Briefings (NotebookLM)                  │
├──────────────────────────────────────────────────────────────────┤
│  AGENTE — KRATOS (Chefe de Máquinas Virtual)                       │
│  • Orquestração de ferramentas  • RAG sobre o grafo                 │
│  • Memória de longo prazo (Recall API)                             │
│  • Motores LLM plugáveis: Claude · Grok AI                         │
├──────────────────────────────────────────────────────────────────┤
│  CONHECIMENTO                                                      │
│  • Grafo (equip.→peça→procedimento→sintoma→causa→ação)            │
│  • Índice vetorial (embeddings)  • Recall API (episódico)          │
├──────────────────────────────────────────────────────────────────┤
│  INGESTÃO                                                          │
│  • Upload  • OCR assistido  • Limpeza/normalização                 │
│  • Extração de entidades  • Construção/atualização do grafo        │
├──────────────────────────────────────────────────────────────────┤
│  DADOS (Supabase: Postgres + pgvector + Storage + Auth)            │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 Papel de cada tecnologia citada

| Tecnologia | Papel no TugGrafos |
|---|---|
| **OCR assistido** | Converte manuais escaneados/fotos em texto pesquisável; "assistido" = o agente revisa termos técnicos e corrige erros de leitura usando dicionário de domínio naval. |
| **Grafos de conhecimento** | Representam *relações* (esta peça pertence a este equipamento; este sintoma indica esta causa). Permitem raciocínio multi-salto que busca por palavra não dá. |
| **Recall API** | Camada de **memória de longo prazo** do agente: lembra documentos já ingeridos, decisões passadas, preferências do navio e contexto entre sessões. |
| **NotebookLM** | Geração de **briefings, resumos e roteiros de áudio** a partir do conjunto de documentos de um equipamento (ex.: "resumo da revisão de 5.000 h"). |
| **Netlify** | Hospedagem do **frontend/PWA** e de funções serverless (edge) para a camada pública; CI/CD a cada commit. |
| **Grok AI** | Um dos **motores LLM** plugáveis do KRATOS (conversação/raciocínio), alternável com Claude conforme custo/latência/disponibilidade. |
| **Assistente de Voz** | STT (fala→texto) + TTS (texto→fala) para operação mãos-livres na praça de máquinas. |
| **KRATOS** | O **agente orquestrador**: decide quais ferramentas chamar, monta o contexto, aplica regras de segurança e exige citação de fonte. |

### 5.3 Fluxo de ingestão (você alimenta → assistente organiza)

```
Documento (PDF/foto/planilha)
   → OCR assistido (texto + layout + tabelas)
   → Normalização (unidades, termos, números de peça)
   → Segmentação (chunks por seção/procedimento)
   → Extração de entidades & relações (LLM + ontologia naval)
   → Upsert no Grafo + Embeddings (pgvector)
   → Registro na Recall API (memória episódica)
   → Pronto para consulta
```

### 5.4 Fluxo de consulta por voz

```
Voz do usuário → STT → KRATOS
   → Recupera contexto (Recall) + busca híbrida (grafo + vetorial)
   → Monta prompt com trechos + citações
   → LLM (Claude/Grok) gera resposta ancorada nas fontes
   → Verifica "tem citação?" (senão: 'não encontrado')
   → TTS → resposta falada + card com fonte (doc, página)
```

---

## 6. Modelo de domínio (ontologia do grafo)

Nós principais:
- **Equipamento** (ex.: Motor de Combustão Principal, Gerador Diesel,
  Purificador, Caldeira, Bomba)
- **Componente/Peça** (ex.: cabeça de cilindro, injetor, rolamento)
- **Procedimento** (ex.: revisão 5.000 h, sangria do sistema)
- **Sintoma** (ex.: alta temp. de gases, vibração anormal)
- **Causa** (ex.: injetor desgastado, filtro obstruído)
- **Ação/Recomendação** (ex.: substituir injetor, limpar filtro)
- **Sobressalente** (número de peça, fabricante, estoque)
- **Documento/Fonte** (manual, boletim, certificado · página)
- **Parâmetro** (torque, folga, pressão, temperatura, óleo)

Arestas de exemplo:
`Equipamento —possui→ Componente`,
`Sintoma —indica→ Causa`,
`Causa —resolvida_por→ Ação`,
`Procedimento —requer→ Sobressalente`,
`Qualquer_nó —fonte→ Documento(página)`.

---

## 7. Stack tecnológica proposta

| Camada | Tecnologia | Observação |
|---|---|---|
| Frontend/PWA | React + Vite, publicado na **Netlify** | Offline-first (service worker, IndexedDB) |
| Voz | Web Speech API + provedor STT/TTS | Fallback on-device para offline |
| Agente | **KRATOS** (orquestrador) | Ferramentas: busca no grafo, vetorial, Recall |
| LLM | **Claude** (padrão) · **Grok AI** (alternativo) | Roteamento por custo/latência/disponibilidade |
| Memória | **Recall API** | Episódica + preferências do navio |
| Resumos | **NotebookLM** | Briefings e áudio por equipamento |
| Banco | **Supabase** (Postgres + **pgvector**) | RAG, grafo relacional, Auth, Storage |
| Grafo | Postgres (tabelas de nós/arestas) ou extensão de grafo | Migração futura p/ banco de grafo dedicado se necessário |
| OCR | Serviço de OCR + pós-processamento por LLM | "Assistido" = correção por dicionário de domínio |
| CI/CD | Git → Netlify (frontend) · Supabase migrations (backend) | Deploy a cada push |

> Nota de integração: NotebookLM e Grok AI não expõem todas as APIs
> publicamente em todas as regiões. O design os trata como **adaptadores
> plugáveis** atrás de interfaces internas, permitindo trocar provedores
> sem reescrever o núcleo.

---

## 8. Diferenciais e guardrails

- **Citação obrigatória**: KRATOS nunca "inventa" — toda afirmação técnica
  carrega `documento + página`. Sem fonte → "não encontrado nos documentos
  carregados".
- **Limites de segurança**: o agente **não** comanda máquinas; apenas
  informa, recomenda e registra. Ações críticas exigem confirmação humana.
- **Offline-first**: núcleo de consulta funciona a bordo; nuvem é para
  enriquecimento e sincronização em porto.
- **Privacidade do armador**: documentos não são usados para treinar
  modelos de terceiros; isolamento por navio/empresa (multi-tenant).
- **Auditável**: histórico completo de perguntas, fontes e versões de
  documento para ISM/classe/PSC.

---

## 9. Roadmap em fases

### Fase 0 — Descoberta (2 semanas)
- Selecionar 1 navio piloto e 1 equipamento (ex.: MCP).
- Coletar ~50 documentos representativos.
- Definir ontologia mínima e métricas de sucesso.

### Fase 1 — MVP de ingestão + busca (4–6 semanas)
- Upload + OCR assistido + normalização.
- Grafo mínimo + busca vetorial híbrida.
- Chat KRATOS com **citação de fonte**.
- Deploy do frontend na Netlify; dados no Supabase.

### Fase 2 — Voz + memória (4 semanas)
- Assistente de voz (STT/TTS) mãos-livres.
- Recall API (memória de longo prazo).
- Roteamento de LLM Claude/Grok.

### Fase 3 — Briefings + grafo visual (4 semanas)
- NotebookLM para resumos/áudio por equipamento.
- Visualizador interativo do grafo.
- Modo offline robusto (sync em porto).

### Fase 4 — Escala + auditoria (contínuo)
- Multi-navio/multi-tenant.
- Painéis de KPI para superintendência.
- Trilha de auditoria e exportação de evidências.

---

## 10. Métricas de sucesso (KPIs)

- **TTA — Time to Answer**: tempo médio para localizar uma especificação
  (meta: de minutos para < 15 s).
- **Cobertura documental**: % de equipamentos com manuais ingeridos.
- **Taxa de citação**: % de respostas com fonte válida (meta: 100%).
- **Precisão verificada**: % de respostas corretas em amostra auditada.
- **Adoção**: consultas/dia por usuário; uso por voz vs. texto.
- **Redução de retrabalho/avaria**: correlação com registros de manutenção.

---

## 11. Riscos e mitigações

| Risco | Impacto | Mitigação |
|---|---|---|
| OCR ruim em manuais antigos/borrados | Dados incorretos | OCR assistido + revisão humana em lote crítico + confiança por trecho |
| Alucinação do LLM | Decisão errada | Citação obrigatória + "não encontrado" + verificação de fonte |
| Conectividade a bordo | Indisponibilidade | Arquitetura offline-first + sync em porto |
| Disponibilidade de APIs (NotebookLM/Grok) | Bloqueio técnico | Adaptadores plugáveis + provedores alternativos |
| Direitos sobre manuais de OEM | Jurídico | Uso interno por navio; respeitar licenças; não redistribuir |
| Segurança de dados do armador | Confidencialidade | Multi-tenant isolado, criptografia, sem treino de terceiros |

---

## 12. Considerações de segurança e conformidade

- **ISM Code / SMS**: TugGrafos como *apoio à decisão*, não substituto de
  procedimentos aprovados.
- **Sociedade classificadora**: respostas rastreáveis à documentação de
  classe.
- **Proteção de dados**: criptografia em repouso e em trânsito; controle de
  acesso por papel (RBAC).
- **Responsabilidade**: o ser humano (Chefe de Máquinas) mantém a decisão
  final; o agente registra recomendações e fontes.

---

## 13. Próximos passos

1. Validar escopo da **Fase 0** (navio e equipamento piloto).
2. Confirmar disponibilidade/credenciais das APIs (Recall, NotebookLM,
   Grok, OCR) e provisionar **Supabase** + **Netlify**.
3. Definir ontologia mínima com um Chefe de Máquinas de referência.
4. Construir o MVP de ingestão + chat com citação (Fase 1).

---

## 14. Glossário

- **Chefe de Máquinas**: oficial responsável pela praça de máquinas do navio.
- **Grafo de conhecimento**: representação de entidades e suas relações.
- **OCR**: reconhecimento óptico de caracteres (imagem → texto).
- **RAG**: geração aumentada por recuperação (resposta ancorada em fontes).
- **PMS**: Planned Maintenance System (sistema de manutenção planejada).
- **PSC**: Port State Control (inspeção de Estado do porto).
- **ISM**: International Safety Management Code.
- **MCP**: Motor de Combustão Principal.
- **STT/TTS**: Speech-to-Text / Text-to-Speech.
- **KRATOS**: agente IA orquestrador "Chefe de Máquinas Virtual".
