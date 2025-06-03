# Resumo: Migração EduPrime para Nuxt 3/Nitro com Directus

## Vantagens

- **Sinergia Tecnológica**  
  - Ecosistema moderno: Directus + Nuxt formam uma stack integrada (Momento de mudar é agora !!!)
  - TypeScript end-to-end: tipos gerados automaticamente a partir do schema Directus  
  - Validação de formulários client-side e server-side consistentes através do mesmo schema (Ganho no server-side)

- **Redução Drástica de Código Backend**  
  - Directus fornece APIs REST e GraphQL sem escrever serviços CRUD  
  - Elimina boa parte do código de serviços customizados  
    - Ex.: `ClassroomService.ts`, `TeacherService.ts`, etc.  
  - Hooks do Directus podem implementar boa parte da lógica de negócio atualmente presente nos serviços  

- **Escalabilidade +1-Município**  
  - Arquitetura serverless do Nitro escala conforme demanda  
  - Multi-tenancy nativo do Directus facilita gerenciar várias escolas/municípios  
    - Substitui a lógica atual de `tenantId` em cada entidade  
  - Modelo de custo “pay-as-you-go”  

- **Performance Aprimorada**  
  - SSR (_Server-Side Rendering_) do Nuxt reduz tempo de carregamento inicial  
  - Caching automático no Nitro diminui carga no banco (Páginas estáticas)
  - GraphQL do Directus permite consultas otimizadas, selecionando apenas campos necessários  

- **Manutenibilidade Superior**  
  - Roteamento automático baseado em arquivos do Nuxt reduz boilerplate (file-based → elimina configuração manual de Vue Router)
    - Elimina a necessidade de manter manualmente rotas para ~40 telas  
  - Painel administrativo pronto do Directus para CRUD  
  - Menos código customizado = menos bugs e manutenção simplificada  

- **Experiência de Desenvolvimento**  
  - Auto-geração de tipos, APIs e documentação a partir do Directus  
  - Monorepo unificado para frontend e eventuais extensões do Directus  

- **Momento Oportuno para Mudança**  
  - Integração com Directus já decidida → adicionar Nuxt/Nitro aproveita a transição  
  - Custo incremental menor do que fazer duas migrações separadas  

---

## Desvantagens

- **Perda de Componentes Ionic Específicos**  
  - Necessidade de recriar componentes customizados  
    - Ex.: `IonSelect` com múltipla seleção usado nos formulários de escola para selecionar recursos como Location, Facility...  
  - Risco de inconsistência visual durante o período de transição  
  - Necessidade de repensar a estratégia mobile-first que o Ionic facilitava nativamente  
    - Ex.: `RegisterFunctionMobile.vue` implementa um bottom sheet específico para dispositivos móveis, que precisaria ser recriado  

- **Rigidez Parcial do Directus**  (Ela existe ?. se sim, segue abaixo meus pontos: )
  - Menos flexibilidade para customizações profundas do modelo de dados  
  - Operações complexas podem exigir extensões personalizadas, com Hooks por exemplo que já podem ser feitas a partir do Nuxt 

- **Overhead para Funcionalidades Simples**  
  - SSR pode ser desnecessário em partes da aplicação que rodariam bem como SPA  (Porém SSR naturalmente é melhor para SEO, confere?)
  - Adiciona complexidade em comparação com uma aplicação puramente client-side  (Ionic puramente client-side)

- **Menor Controle Direto sobre o Backend**    
  - Adaptação ao paradigma de coleções e Hooks do Directus pode exigir ajustes de mentalidade  

---

## Riscos

- **Complexidade da Migração Dupla**  
  - Curva de aprendizado simultânea (Directus + Nuxt)  
  - Coordenação entre migração de dados (Directus) e migração de interface (Nuxt)  

- **Regressões em Formulários Complexos**  
  - Lógicas específicas podem ser difíceis de replicar no Nuxt  
  - Exemplo concreto: `RegisterClassroom.vue` implementa lógica para habilitar botões usando comparação profunda de objetos
    ```js
    // Lógica atual que compara estado atual com estado inicial para detectar mudanças
    const enableSaveButton = computed(() => {
      return JSON.stringify(formValues) !== JSON.stringify(initialFormValues)
    })
    ```
  - Comportamentos sutis de validação ou interação podem ser perdidos  

- **Integração Capacitor/Mobile**  
  - Configuração adicional para manter recursos nativos (câmera, geolocalização) no Nuxt  
  - Possíveis problemas com bottom sheets e componentes nativos que existiam no Ionic  
  - Exemplo: `RegisterFunctionMobile.vue` usa breakpoints específicos do Ionic
    ```html
    <ion-modal 
      :is-open="isOpen"
      :initial-breakpoint="0.7"
      :breakpoints="[0, 0.25, 0.5, 0.7, 1]" 
      @didDismiss="handleDidDismiss">
    </ion-modal>
    ```  

- **Múltiplos enums**  
  - Formulários como o de registro de escolas usa múltiplos enums do Prisma (Location, Usage, Facility...) que precisarão ser cuidadosamente mapeados na nova implementação

- **Customizações Específicas do Domínio**  
  - Regras de negócio complexas podem não se encaixar facilmente no modelo do Directus  
  - Pode haver demanda por endpoints customizados para casos muito específicos  

---

# Plano de Migração: EduPrime para Nuxt 3/Nitro com Directus

## Visão Geral

A migração será conduzida de forma incremental, priorizando primeiro componentes de baixa complexidade e avançando para formulários e fluxos críticos de negócio. Directus já fornecerá CRUD básico e painel administrativo; portanto, focaremos em migrar para Nuxt/Nitro apenas o que não está coberto por Directus.

---

## Mapeamento de Componentes para Migração ao Nuxt 3

### 1: Baixa Complexidade 

#### Páginas de Visualização
- **`src/views/NotificationsPage.vue`**  
  - Página simples que consumirá API Directus  
  - Estrutura quase puramente visual, poucas dependências de lógica  

- **`src/views/ProfilePage.vue`**  
  - Exibição de perfil de usuário, integrando autenticação Directus  
  - Layout simples, poucos componentes Ionic  

- **`src/views/CalendarPage.vue`**  
  - Visualização de calendário independente  
  - Beneficia-se imediatamente de SSR para carregamento rápido  

#### Componentes Utilitários
- **`src/components/`** (componentes de UI reutilizáveis)  
  - Cards, loaders e botões personalizados (substituir Ionic por componentes Nuxt/Tailwind)  
  - Boa oportunidade para criar biblioteca base de componentes Nuxt  

---

### 2: Complexidade Intermediária

#### Visualizações de Dashboard
- **`src/views/HomePage.vue`**  
  - Dashboard principal com diversos cards e gráficos  
  - Consumirá múltiplas APIs Directus para montar informações  
  - Ideal para ilustrar benefícios do Nuxt SSR  

---

### 3: Formulários Moderados

#### Componentes de Pré-Matrícula
- **`src/modules/pre-enrollment/*`** (componentes de formulário)  
  - Formulários de pré-matrícula com validações intermediárias  
  - Substituir `IonSelect` por componente Vue equivalente (por ex., `Multiselect` ou `VueSelect`)  

#### Componentes de cadastro
- **`src/modules/school-management/*`** (exceto `Register'Mobile'`)  
  - Formulários  
  - Componentes de listagem e seleção

---

### 4: Alta Complexidade

#### Formulários Complexos
- **`src/modules/school-management/RegisterClassroom.vue`**  
  - Lógica de habilitação de botão usando comparação profunda via `JSON.stringify`  
  - Manipulação de arrays de dias da semana e disciplinas  
  - Requer atenção especial para clonar estado inicial e replicar funcionalidade “dirty deep” no Nuxt  

- **Formulários de Infraestrutura Escolar**  
  - Localizados em `src/modules/school-management/`  
  - Múltiplas seções com enums do Prisma (Location, Usage, etc.)  
  - **Selects múltiplos complexos**:
    ```html
    <!-- Exemplo de select múltiplo para infraestrutura -->
    <ion-select multiple="true" v-model="formValues.infrastructure.location">
      <ion-select-option
        v-for="option in locationOptions"
        :key="option.value"
        :value="option.value">
        {{ option.label }}
      </ion-select-option>
    </ion-select>
    ```
  - Necessário recriar dropdowns que suportem seleção múltipla com mesma usabilidade e performance  

#### Componentes Mobile Específicos
- **`src/modules/institution/RegisterFunctionMobile.vue`**  
  - Bottom sheet com breakpoints específicos do Ionic:
    ```html
    <ion-modal 
      :is-open="isOpen"
      :initial-breakpoint="0.7"
      :breakpoints="[0, 0.25, 0.5, 0.7, 1]" 
      @didDismiss="handleDidDismiss">
    </ion-modal>
    ```
  - Detecta dispositivo por largura de tela (< 768px)  
  - Componente crítico para experiência mobile, precisa ser recriado com `<Teleport>` e `<transition>` ou biblioteca equivalente (por ex., `vue-final-modal`)

---

### 5: Complexidade Máxima

#### Módulos de Gestão Pedagógica
- **`src/modules/teacher-journey/*`** (37 componentes)  
  - Fluxos completos de gestão de professores  
  - Formulários de horários, planos de aula, dependem de dados interligados  
  - Alta interdependência entre componentes, demandando roteamento e stores Pinia bem definidos  

#### Fluxos Críticos de Negócio
- **`src/modules/school-management/`**  
  - Componentes de quadro de horários !!!
  - Formulários com validações específicas (Como turmas e séries)
  - Dependências múltiplas que exigirão endpoints Nitro customizados além do Directus  

---

## Etapas de Migração

### 1: Preparação e Componentes de Baixa Complexidade

1. **Configurar Directus com o schema existente**  
   - Importar tabelas e relacionamentos no Admin App  
   - Ajustar enums e permissões básicas (papéis: Admin, Secretaria, Professor, Aluno)  

2. **Criar projeto Nuxt 3 básico**  
   - Adicionar módulos: `@pinia/nuxt`, `@nuxtjs/tailwindcss`, `@nuxtjs/axios`, `@nuxtjs/pwa`  
   - Configurar `runtimeConfig.public.directusBase` apontando para a API Directus  

3. **Migrar Nível 1 de componentes**  
   - `NotificationsPage.vue`, `ProfilePage.vue`, `CalendarPage.vue`  
   - Criar componentes utilitários em `components/` (cards, loaders, botões)  
   - Garantir SSR em páginas de visualização simples  

---

### 2: Módulos de Complexidade Intermediária

1. **Migrar `HomePage.vue` e `CalendarPage.vue`**  
   - `HomePage.vue`: consumir múltiplas APIs Directus para montar dashboard com cards e gráficos (usar Echarts via `<client-only>`)  
   - `CalendarPage.vue`: renderizar calendário, integrando com dados de eventos via Directus  

2. **Criar wrappers TypeScript para as APIs Directus**  
   - Substituir `SeriesService.ts`, `StageService.ts` por chamadas ao SDK ou `$fetch` direto  
   - Garantir tipagem automática a partir do schema Directus  

---

### 3: Formulários Moderados e Altamente Complexos

1. **Migrar componentes (Pré-Matrícula...)**  
   - Criar formulário de pré-matrícula, com validações Nuxt/VeeValidate  
   - Substituir `IonSelect multiple` por componente de seleção múltipla em Vue (por ex., `vue-multiselect`)  
   - Migrar formulários, integrando com APIs de Directus  

2. **Migrar componentes (Formulários mais complexos e Mobile)**  
   - **`RegisterClassroom.vue`**:
     - Garantir clone profundo do estado inicial (`initialFormValues`) antes de tornar reativo  
     - Utilizar `lodash.isEqual` ou similar para comparação profunda e habilitação de botão "Salvar"
   - **Formulários com ligação com diversos enums e/ou relacionamentos (Mais de uma tabela envolvida)r**:
     - Refatorar selects múltiplos para usar componentes nativos ou bibliotecas (e.g., `VueSelect`)  (Semelhante ao mencionado anteriormente)
   - **`RegisterFunctionMobile.vue`**:
     - Recriar bottom sheet usando `<Teleport>` e `<transition>` ou biblioteca de modais mobile-friendly  
     - Ajustar detecção de dispositivo via composição de reatividade (`useWindowSize()` ou `@vueuse/core`)  

---

### 4: Fluxos Críticos e Integrações Avançadas

1. **Migrar componentes**  
   - Refatorar componentes de `teacher-journey` e `student-management` para consumir dados do Directus  
   - Implementar stores Pinia bem estruturados para gerenciar estado compartilhado (professores, alunos, matrículas)  

2. **Criar endpoints Nitro para lógica que ultrapassa Directus**  
   - **Quadro de Horários** !!!`  
   - PLUS graças a Directus = **Relatórios Avançados**: gerar CSV/PDF de notas e frequência em rotas Nitro, usando dados agregados do Directus  

3. **Integração Capacitor**  
   - Configurar Capacitor para build mobile, garantindo acesso a plugins nativos (câmera, geolocalização...)  
   - Implementar estratégia de cache de dados (IndexedDB ou `localforage`) para funcionalidades offline 
---

### 5: Finalização

1. **Multi-tenancy e Escalabilidade +1 Município**  
   - Validar que o Directus aplica isolamento de dados conforme os papéis e coleções por município  
   - Configurar caching avançado no Nitro (Cache-Control headers ou Redis)  
   - Monitorar erros e performance, recolhendo feedback dos usuários para ajustes *

---
