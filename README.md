# AndHiring PRO

> Triagem e scoring de candidatos com IA — do currículo à shortlist em minutos.

**Versão:** 3.3 | **Status:** Produto funcional em busca do primeiro cliente pagante
**Período de desenvolvimento:** 18 de março de 2026 → 04 de abril de 2026 (17 dias)

---

## O que é

AndHiring PRO é um SaaS de recrutamento B2B que usa Inteligência Artificial para automatizar a triagem, scoring e shortlist de candidatos.

O recrutador define a vaga via chat inteligente e envia os currículos dos candidatos (PDF, DOCX, texto colado ou link do LinkedIn). A IA analisa cada perfil contra os requisitos da vaga, gera um score de 0 a 100, produz uma análise executiva com pontos fortes, preocupações e recomendação de avanço — e persiste tudo no banco de dados.

---

## A dor que resolve

Recrutadores gastam entre 4 e 8 horas por vaga apenas na triagem manual de currículos — lendo, comparando e tentando priorizar candidatos sem critério objetivo.

AndHiring PRO reduz esse processo para menos de 30 minutos: o recrutador define a vaga uma vez, sobe os perfis e recebe uma shortlist ranqueada com justificativas por candidato, pronta para decisão.

**Para quem:**
- Recrutadores solo (dor mais aguda, decisão mais rápida)
- Pequenas consultorias de RH (2–10 pessoas)
- Empresas com RH interno processando mais de 5 vagas/mês

---

## Funcionalidades

### Fluxo 1 — Agente de Definição de Vaga
- Chat interativo que conduz o recrutador por perguntas estruturadas até completar o perfil da vaga
- Extrai automaticamente: cargo, hard skills, soft skills, requisitos, must-haves, senioridade, localização e modelo de trabalho
- Suporte a scraping de URL: detecta links de plataformas como Solides e Gupy e extrai o texto da vaga automaticamente
- Painel lateral que atualiza os requisitos em tempo real à medida que o agente conversa
- Salva a vaga completa no Supabase ao final do chat (`complete: true`)
- Sessão persistente: `session_id` e `job_id` mantidos no `localStorage` entre reloads

### Fluxo 2 — Análise de Candidatos
- Aceita PDF, DOCX, texto colado e URLs de perfil
- Extração de texto de PDFs diretamente no browser via pdf.js (sem servidor)
- Análise individual por IA para cada candidato com output estruturado:
  - `score` (0–100)
  - `executive_summary` (análise para nível C-Level)
  - `highlights` (pontos fortes com evidências do currículo)
  - `concerns` (desalinhamentos com diagnóstico)
  - `hard_skills` e `soft_skills` identificados
  - `recommendation`: avançar / avançar com ressalvas / não avançar
  - `conclusion` (decisão final com sugestão de reposicionamento)
  - `outreach_angle` (ângulo personalizado de abordagem)
- Ranking automático por score no frontend
- Persistência de todos os candidatos analisados na tabela `candidates` do Supabase

### Fluxo 3 — Shortlist
- Botão de shortlist por candidato direto no ranking
- Salva o candidato selecionado no Supabase com todos os campos de análise
- UI de shortlist com score, pontos fortes, atenções, skills e sistema de estrelas (1–5)
- Exportação da shortlist como relatório PDF diretamente pelo browser

### Autenticação e Multi-cliente
- Login via magic link (Supabase Auth) — sem senha, sem fricção
- Sessão identificada por `client_id` (UUID do usuário autenticado)
- Todas as vagas, candidatos e shortlist isolados por cliente no banco de dados
- Badge com e-mail do usuário logado + botão de logout na navbar

---

## Stack Técnica

| Camada | Ferramenta | Detalhe |
|---|---|---|
| Frontend | GitHub Pages | HTML / CSS / JS puro — sem framework |
| Orquestração | n8n self-hosted (Railway) | Template "N8N (w/ workers)": Postgres + Redis + Worker + Primary |
| IA | Groq — `llama-3.3-70b-versatile` | Agente de vaga + análise de candidatos |
| Banco de dados | Supabase (PostgreSQL, região SP) | Tabelas: `jobs_v2`, `candidates`, `shortlist` |
| Autenticação | Supabase Auth | Magic link por e-mail (OTP sem senha) |
| Extração de PDF | pdf.js (client-side) | Sem dependência de servidor para leitura de PDFs |

---

## Técnicas de IA Utilizadas

| Técnica | Onde | Impacto |
|---|---|---|
| **Conversational Agent** | Fluxo 1 — Agente de Vaga | Chat estruturado que extrai requisitos em loop até completar o perfil |
| **Chain of Thought (CoT)** | Fluxo 2 — Scoring | IA mapeia requisitos e busca evidências antes de atribuir score — reduz alucinações |
| **Arquitetura ReAct** | Fluxo 2 — IF Dados Suficientes | IA raciocina se os dados são suficientes e age consultando histórico no Supabase antes de prosseguir |
| **JSON-mode prompting** | Ambos os fluxos | Output sempre estruturado e parseável — sem texto livre fora do JSON |
| **LLM-as-a-Judge** *(planejado)* | Fluxo 2 — pós-análise | Segundo agente avalia a qualidade da análise produzida pelo agente principal |
| **Tree of Thoughts (ToT)** *(planejado)* | Fluxo 2 — Scoring | 3 perspectivas independentes (técnica, fit, potencial) antes do score final |
| **Self-Consistency** *(planejado)* | Fluxo 2 — zona cinzenta | Scoring 3x para candidatos entre 55–75, usa mediana — reduz variância |

---

## Arquitetura dos Fluxos n8n

```
FLUXO 1 — Agente de Vaga  (/webhook/agente-vaga)
─────────────────────────────────────────────────
Webhook → Detecta URL?
            ↓ SIM → HTTP Request (scraping) → Code (extrai texto) → AI Agent
            ↓ NÃO → AI Agent
          AI Agent → Code (parse JSON) → IF (complete?)
                                           ↓ true  → Supabase INSERT jobs_v2 → Respond to Webhook
                                           ↓ false → Respond to Webhook


FLUXO 2 — Análise de Candidatos  (/webhook/analyze-profiles)
─────────────────────────────────────────────────────────────
Webhook → Get a row (jobs_v2) → Code (prepara candidatos) → AI Agent
        → Wait (5s) → Code (parse JSON) → Supabase INSERT candidates
        → Aggregate → Respond to Webhook


FLUXO 3 — Shortlist  (/webhook/shortlist)
──────────────────────────────────────────
Webhook → Supabase INSERT shortlist → Respond to Webhook
```

---

## Banco de Dados

### `jobs_v2` — Vagas
| Coluna | Tipo |
|---|---|
| id | UUID (PK) |
| session_id | TEXT |
| client_id | UUID |
| title | TEXT |
| persona_ideal | TEXT |
| hard_skills | JSONB |
| soft_skills | JSONB |
| requirements | JSONB |
| must_haves | JSONB |
| seniority | TEXT |
| location | TEXT |
| work_model | TEXT |
| complete | BOOLEAN |
| created_at | TIMESTAMP |

### `candidates` — Candidatos Analisados
| Coluna | Tipo |
|---|---|
| id | UUID (PK) |
| job_id | UUID |
| session_id | TEXT |
| client_id | UUID |
| name | TEXT |
| current_role | TEXT |
| location | TEXT |
| score | INTEGER |
| summary | TEXT |
| highlights | JSONB |
| concerns | JSONB |
| hard_skills | JSONB |
| soft_skills | JSONB |
| recommendation | TEXT |
| conclusion | TEXT |
| outreach_angle | TEXT |
| created_at | TIMESTAMP |

### `shortlist` — Candidatos Aprovados
| Coluna | Tipo |
|---|---|
| id | UUID (PK) |
| job_id | UUID |
| client_id | UUID |
| candidate_name | TEXT |
| score | INTEGER |
| highlights | JSONB |
| concerns | JSONB |
| hard_skills | JSONB |
| soft_skills | JSONB |
| outreach_angle | TEXT |
| starred | INTEGER |
| created_at | TIMESTAMP |

---

## URLs de Produção

| Recurso | URL |
|---|---|
| Frontend | https://andymendes07.github.io/AndHiring-PRO---Agente-de-Vaga/ |
| Backend (n8n Railway) | https://primary-production-0fd8.up.railway.app |
| Webhook — Agente de Vaga | `/webhook/agente-vaga` |
| Webhook — Análise de Candidatos | `/webhook/analyze-profiles` |
| Webhook — Shortlist | `/webhook/shortlist` |

---

## Histórico de Desenvolvimento

| Período | Sessões | O que foi construído |
|---|---|---|
| 18–26/03/2026 | 1–4 | Fluxo original (PhantomBuster + EnrichLayer + Anthropic API), pivô para modelo "recrutador traz candidatos", implementação dos 3 fluxos, correções de CORS, session persistence, duplo disparo e scoring |
| 26/03/2026 | 5 | Chain of Thought no scoring, arquitetura ReAct com IF Dados Suficientes |
| 30/03/2026 | 6 | Migração do n8n Cloud (trial expirado) para Railway self-hosted, reconfiguração de credenciais, validação end-to-end completa |
| 04/04/2026 | 7 | Autenticação via magic link, separação multi-cliente com `client_id`, persistência de candidatos na tabela `candidates`, controle de rate limit Groq, badge de recomendação no modal, fix do campo `persona_ideal` |

**Total:** 17 dias do zero ao produto funcional com autenticação, banco de dados e análise de IA em produção.

---

## Próximos Passos

| Prioridade | Tarefa |
|---|---|
| 🔴 Alta | Primeiro cliente pagante — produto está funcional |
| 🔴 Alta | Notificação browser quando análise termina (Notification API) |
| 🟡 Média | LGPD: banner de consentimento + política de privacidade |
| 🟡 Média | LLM-as-a-Judge: segundo agente validando qualidade das análises |
| 🟡 Média | Tree of Thoughts no scoring: 3 perspectivas antes do score final |
| 🟡 Média | Exportação de relatório por e-mail |
| 🟢 Baixa | Self-Consistency para candidatos na zona cinzenta (55–75) |
| 🟢 Baixa | Segunda chave de API Groq para controle de rate limit |

---

## Criado por

**Anderson Mendes Silva**
silva.andersonmendes@gmail.com
