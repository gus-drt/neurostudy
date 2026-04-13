Documento de Arquitetura e Regras de Negócio (NeuroStudy)
Objetivo: Servir como contexto base (Master Prompt) para agentes de IA desenvolverem o sistema.

1. Stack Tecnológica (Recomendada)
Frontend: React (Next.js App Router) - React, TailwindCSS, Lucide Icons.

Backend / BaaS: Supabase (PostgreSQL, Auth, RLS Policies).

Spaced Repetition: ts-fsrs (Implementação TS do algoritmo FSRS).

AI Integration: Vercel AI SDK (conectado ao Gemini 1.5 Flash ou gpt-4o-mini).

Speech-to-Text: Web Speech API (nativo do browser).

2. Modelagem do Banco de Dados (Supabase / PostgreSQL)
A estrutura deve ser estritamente relacional.

Tabela: decks
Agrupa os cards por assunto.

id (uuid, PK)

user_id (uuid, FK para auth.users)

title (text)

created_at (timestamp)

Tabela: cards
Armazena o conteúdo estático.

id (uuid, PK)

deck_id (uuid, FK para decks)

question (text)

answer (text)

tags (text[])

Tabela: fsrs_states (O Motor da Memória)
Armazena o estado dinâmico do card para um usuário. (Baseado na especificação FSRS).

card_id (uuid, PK/FK)

user_id (uuid, PK/FK)

due (timestamp) - Data/hora em que o card deve ser revisado.

stability (float) - Estabilidade da memória (S).

difficulty (float) - Dificuldade do card (D).

elapsed_days (int) - Dias desde a última revisão.

scheduled_days (int) - Dias agendados até a próxima revisão.

reps (int) - Quantidade de revisões totais.

lapses (int) - Quantidade de vezes que o usuário esqueceu o card.

state (enum: 0=New, 1=Learning, 2=Review, 3=Relearning)

last_review (timestamp)

Tabela: review_logs (Para Analytics/Mapa de Calor)
id (uuid, PK)

card_id (uuid, FK)

user_id (uuid, FK)

rating (int: 1=Again, 2=Hard, 3=Good, 4=Easy)

review_duration_ms (int) - Tempo gasto pensando.

created_at (timestamp)

3. Regras de Negócio Essenciais
A. Algoritmo de Revisão (FSRS)
Sempre que a tela de "Revisão" for aberta, consultar a tabela fsrs_states onde user_id = atual e due <= now().

Quando o usuário avaliar um card (Again, Hard, Good, Easy), importar os dados atuais de fsrs_states para o objeto do ts-fsrs, aplicar a nota, e salvar o novo objeto due, stability, difficulty, etc., atualizando a linha no banco e registrando um review_log.

B. Pipeline de Geração de Cards (LLM)
Regra de Fatiamento: Textos maiores que 3000 palavras devem ser divididos em chunks de ~1000 palavras no frontend antes do envio para a API.

Prompt Restrito: O agente gerador deve obrigatoriamente retornar um JSON array com [{ question: "", answer: "", tags: [] }]. O prompt de sistema deve proibir introduções do tipo "Aqui estão seus cards".

Aprovação Humana: O retorno do LLM não vai direto para o banco de dados. Ele vai para um estado temporário em memória (React State). O usuário deve clicar em "Salvar" ou deletar/editar os cards gerados antes do INSERT no Supabase.

C. A "Blurting Zone" e Validação Semântica
O botão "Mostrar Resposta" fica oculto/desabilitado. O usuário deve usar o input de texto ou o botão de microfone (Web Speech API).

Se o usuário usar STT (voz), o texto transcrito preenche a textarea.

Avaliação da IA (Opcional): Após submeter o "blurt", chamar uma server action rápida que compara a textarea com a answer oficial do card e retorna um feedback: {"match_score": 80, "feedback": "Você lembrou o principal, mas esqueceu a definição de X."}.

4. Diretrizes de UX (Neurodivergência)
Redução de Carga Cognitiva: Usar cores neutras/pasteis (ex: slate-50 para fundo). Animações apenas suaves (duration-300).

Feedback Positivo: Toda vez que um deck for zerado, exibir mensagem clara incentivando pausa (Modo Difuso).

Intercalação: A query que busca os cards vencidos (due <= now()) deve aplicar um ORDER BY random() caso pertençam a decks diferentes, para forçar a mudança de contexto cerebral (Interleaving).