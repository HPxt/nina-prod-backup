# Documentação Técnica para Contribuintes — Nina 1.0

> Documento de onboarding técnico e funcional para manutenção, evolução e governança do projeto.

## 1) Contexto do produto

A **Nina 1.0** é uma plataforma interna para gestão de liderança, acompanhamento de equipe e execução de rituais de gestão na 3A RIVA. O foco do sistema é garantir aderência operacional em interações obrigatórias (1:1, Índice de Risco, N3, PDI, N2 e Índice de Qualidade), além de fornecer visão analítica para líderes, diretores e administradores. 

A visão funcional principal está no README e no blueprint do projeto. 

---

## 2) Stack e arquitetura

### Front-end
- **Next.js 15** (App Router), React 18, TypeScript.
- UI com componentes próprios + Radix + Tailwind.
- Estado distribuído por hooks (Firebase, cache, configuração, permissões).

### Back-end (BaaS + Functions)
- **Firebase Firestore** como banco primário.
- **Firebase Auth** com login Google e controle por domínio corporativo.
- **Cloud Functions** para automações (ranking, backup, setup admin, OAuth Google, migrações).

### Pastas-chave
- `src/app`: rotas/páginas do sistema (login, dashboard e módulos).
- `src/components`: componentes visuais e formulários de domínio.
- `src/lib`: tipos, schemas, utilitários, exportação de dados.
- `src/hooks`: regras transversais (cache, permissões, preload).
- `functions/src`: funções administrativas e gatilhos server-side.
- `docs/`: documentos funcionais e de métricas.

---

## 3) Modelo de domínio e dados

As entidades de negócio principais estão tipadas em `src/lib/types.ts`:

- **Employee**: cadastro mestre do colaborador/líder/diretor/admin.
- **Interaction**: registro de interações (1:1, feedback, N3, risco, N2, qualidade etc.).
- **PDIAction** e **Diagnosis**: plano e diagnóstico de desenvolvimento.
- **Project / ProjectInteraction / Premissas**: módulos de projetos independentes e projeções.

### Campos críticos em `Employee`
- `role`: determina perfil-base (`Colaborador`, `Líder`, `Líder de Projeto`, `Diretor`).
- `isDirector`, `isAdmin`: permissões elevadas.
- `isUnderManagement`: indica se entra em métricas e painéis de acompanhamento.
- `leaderId`: vínculo hierárquico para cálculo de aderência.

### Organização Firestore (visão prática)
- `employees/{employeeId}`
  - `interactions/{interactionId}`
  - `pdiActions/{pdiActionId}`
  - `meetings/{meetingId}`
  - `premissas/{premissaId}`
- `projects/{projectId}/interactions/{interactionId}`
- `leaderRankings/{leaderId}`
- `configs/{configId}`

---

## 4) Regras de negócio essenciais

## 4.1 Controle de acesso

1. Login via Google e validação de usuário em `employees`.
2. Acesso ao dashboard é permitido para perfis de liderança/administração; colaborador puro não entra.
3. Regras de Firestore estão autenticadas por `request.auth != null`, mas **boa parte da autorização fina está no frontend** (importante para contribuintes reforçarem no backend quando necessário).

## 4.2 Agendas e expectativas de rituais

As periodicidades usadas no cálculo de aderência/ranking estão centralizadas em múltiplos módulos (`ranking`, `dashboard v2`, `n2 form`, `admin`):

- **N3 Individual por segmento**:
  - Alfa: 4/mês
  - Beta: 2/mês
  - Senior: 1/mês
- **1:1**: meses 3, 6, 9, 12 (índices 2,5,8,11)
- **PDI**: janeiro e julho
- **Índice de Risco**: mensal

## 4.3 Fórmula do ranking de líderes

A lógica base considera, para cada liderado em gestão:
- Interações esperadas no período (N3 + 1:1 + risco + PDI).
- Interações concluídas no período (com limite para não exceder o esperado em N3).
- **Aderência (%)** = `totalCompleted / totalRequired * 100`.
- **Bônus opcional**: `+3%` a cada bloco de 10 interações concluídas (`Math.floor(totalCompleted / 10) * 3`), controlado por `rankingBonusEnabled`.
- **TotalScore** = `adherenceScore + bonusPercentage`.

## 4.4 N2 Individual (diretor -> líder)

Existe tabela de frequência por líder (semanal/quinzenal/mensal), com fallback mensal quando não há match exato de nome. A comparação usa normalização de acento e busca por primeiro/último nome para reduzir erro de cadastro.

## 4.5 Índice de Risco

Score ponderado por categoria:
- Pesos: Performance(2), Atendimento(1), Remuneração(3), Desenvolvimento(1), Processos(1), Engajamento/Presença(2).
- Mapeamento: `red = +1`, `neutral = 0`, `green = -1`.
- Resultado final = soma ponderada (quanto maior, pior risco).

## 4.6 Índice de Qualidade

Usa as mesmas famílias de peso, porém invertendo semântica:
- `red = -1`, `neutral = 0`, `green = +1`.
- Resultado final favorece score positivo como melhor qualidade.

## 4.7 Triggers e automações

No create de interação:
- atualiza ranking agregado (`leaderRankings`);
- cria evento de calendário (quando aplicável);
- pode disparar e-mail de N3 ao assessor quando `type === "N3 Individual"` e `sendEmailToAssessor === true`.

No write de PDI:
- também aciona atualização de ranking agregado.

## 4.8 Backup e exportação

- Frontend possui exportação CSV/PDF de histórico por colaborador.
- Cloud Functions possuem endpoints administrativos para backup manual/listagem/restauração (com proteção por claim admin).

---

## 5) Responsabilidades por módulo (guia para manutenção)

### `src/app/login/page.tsx`
Responsável por:
- autenticação inicial;
- validação de acesso por perfil;
- fluxo de autorização Google OAuth adicional (popup) para integrações.

### `src/app/dashboard/v2/page.tsx`
Responsável por:
- painel operacional de aderência por colaborador;
- filtros por equipe, status, período e tipo de interação;
- cálculo de status por obrigação e período.

### `src/app/dashboard/ranking/page.tsx`
Responsável por:
- ranking consolidado por líder;
- cálculo percentual + bônus;
- uso de cache local para reduzir consultas repetidas.

### `src/components/individual-tracking-content.tsx`
Responsável por:
- timeline individual;
- criação de interações;
- integração com envio opcional de e-mail no N3;
- persistência de `riskScore` no registro do colaborador.

### `src/components/risk-assessment-form-dialog.tsx`
Responsável por:
- UI/form de avaliação de risco;
- cálculo local do score ponderado.

### `src/components/quality-index-form-dialog.tsx`
Responsável por:
- UI/form de avaliação de qualidade;
- cálculo local do score ponderado invertido.

### `src/components/n2-individual-form.tsx`
Responsável por:
- coleta de N2;
- cálculo automático de “nota ranking” do líder no mês.

### `functions/src/triggers.ts`
Responsável por:
- orquestrar automações em eventos de Firestore;
- executar side-effects confiáveis (ranking, calendário, e-mail N3).

### `firestore.rules`
Responsável por:
- gate mínimo de autenticação;
- proteção parcial de coleções;
- currently permissivo em subcoleções (trade-off explícito para produtividade/UI testing).

---

## 6) Segurança, governança e dívida técnica

### Pontos fortes atuais
- Projeto já evoluiu de regras totalmente abertas para exigir autenticação (`isSignedIn`).
- Funções críticas administrativas exigem `custom claim isAdmin`.
- Tokens OAuth podem ser migrados/criptografados por funções de migração.

### Riscos e atenção para contribuintes
1. **Autorização muito concentrada no frontend** para várias operações de escrita em Firestore.
2. **Listas hardcoded de e-mails admin/teste** em partes do frontend.
3. **Duplicação de regras de periodicidade** em vários arquivos (risco de drift).
4. **Comparação por nome** em frequência de N2 (frágil para cadastro inconsistente).

### Melhorias recomendadas (ordem sugerida)
1. Centralizar regras de periodicidade e pesos em módulo único versionado.
2. Endurecer `firestore.rules` por função/escopo, reduzindo permissões amplas.
3. Reduzir hardcode de admin/teste para configuração central (`configs`).
4. Migrar regras sensíveis de autorização para backend/functions quando viável.

---

## 7) Fluxo de desenvolvimento local

### Pré-requisitos
- Node.js compatível com Next 15 (projeto usa `package-lock`, e functions exigem Node 22).
- Firebase CLI configurado para emuladores/deploy.

### Comandos principais (app web)
- `npm run dev` — sobe app local na porta 9002.
- `npm run build` — build de produção.
- `npm run lint` — lint Next.
- `npm run typecheck` — verificação TypeScript.
- `npm run test:run` — suíte vitest em modo CI.

### Comandos principais (functions)
- `cd functions && npm run build` — compila functions.
- `cd functions && npm run serve` — emulador local functions.
- `cd functions && npm run deploy` — build + deploy functions.

---

## 8) Boas práticas para novos contribuidores

1. **Antes de alterar cálculo**, localizar TODAS as cópias da regra (ranking, admin, n2, dashboard v2).
2. **Evitar alterar tipagem sem atualizar forms + export + timeline**.
3. **Toda mudança de autorização exige revisão conjunta**:
   - login;
   - hooks de permissão;
   - firestore rules;
   - cloud functions.
4. **Adicionar/atualizar testes** de schema e utilitários em mudanças de regra.
5. **Documentar alterações de negócio** em `docs/` no mesmo PR.

---

## 9) Checklist de PR (sugestão)

- [ ] Regra de negócio atualizada e validada com produto.
- [ ] Tipos de domínio coerentes (`src/lib/types.ts`).
- [ ] Sem regressão nos cálculos de aderência/ranking.
- [ ] Permissões revisadas (`frontend + rules + functions`).
- [ ] Testes executados localmente.
- [ ] Documentação funcional/técnica atualizada.

---

## 10) Glossário rápido

- **Aderência**: razão entre rituais esperados e realizados.
- **N3 Individual**: ritual de acompanhamento operacional por assessor.
- **N2 Individual**: ritual diretor ↔ líder.
- **PDI**: plano de desenvolvimento individual.
- **Índice de Risco**: score ponderado onde positivo representa maior risco.
- **Índice de Qualidade**: score ponderado onde positivo representa melhor qualidade.

---

## 11) Referências internas úteis

- `README.md`
- `docs/blueprint.md`
- `docs/metricas-aderencia.md`
- `firestore.rules`
- `src/lib/types.ts`
- `src/app/dashboard/ranking/page.tsx`
- `src/app/dashboard/v2/page.tsx`
- `src/components/n2-individual-form.tsx`
- `src/components/risk-assessment-form-dialog.tsx`
- `src/components/quality-index-form-dialog.tsx`
- `functions/src/triggers.ts`
- `functions/src/index.ts`

