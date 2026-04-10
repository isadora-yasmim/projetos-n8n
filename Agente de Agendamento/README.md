# 📅 Workflow de Agendamento - Sobrevidas (n8n)

## 📌 Visão Geral

Este workflow implementa um **assistente virtual de agendamento de consultas via Telegram**, utilizando IA para conduzir a conversa com o paciente e realizar o agendamento de forma automatizada.

Ele suporta:

* Mensagens de texto
* Mensagens de áudio (com transcrição)
* Integração com banco de dados (PostgreSQL)
* Controle de fluxo conversacional com regras bem definidas

---

## 🚀 Fluxo Geral

```text
Telegram → Identificação do tipo → (Texto | Áudio | Outros)
        → Padronização → Agente IA → Tools (DB)
        → Processamento → Envio das mensagens
```

---

## 📥 Entrada da Mensagem

### 🔹 Telegram Trigger

Responsável por iniciar o fluxo ao receber mensagens no bot.

### 🔹 Switch (Identifica o tipo da mensagem)

Classifica a entrada em:

* **Texto** → segue direto
* **Áudio** → vai para transcrição
* **Outros (imagem, vídeo, etc.)** → mensagem de erro padrão

### 🔹 Mensagem não suportada

```text
Desculpe, não consigo processar a sua mensagem, por favor me mande apenas textos ou áudios
```

---

## 🎙️ Processamento de Áudio

Caso o usuário envie áudio:

1. **Get Arquivo de Áudio**
   Obtém o caminho real do arquivo via API do Telegram

2. **Download Áudio**
   Baixa o arquivo em formato binário

3. **Transcribe a recording (OpenAI)**
   Converte áudio → texto (Whisper, idioma pt-BR)

---

## 🔄 Padronização dos Dados

Node: **Set (Padronização dos dados)**

Cria um formato padrão para o agente:

```json
{
  "mensagem_usuario": "texto ou transcrição",
  "session_id": "chat_id",
  "origem": "texto | audio"
}
```

---

## 🤖 Agente de Agendamento

### 🔹 Modelo

* GPT (OpenAI)

### 🔹 Memória

* Buffer com janela de contexto das últimas mensagens

### 🔹 AI Agent

Responsável por:

* Interpretar a mensagem do usuário
* Seguir o fluxo conversacional obrigatório
* Executar tools (queries no banco)

---

## 🧠 Fluxo Conversacional (Obrigatório)

O agente segue estritamente estas etapas:

1. **Apresentação**
   Se apresenta e solicita o nome completo

2. **Buscar paciente**
   Tool: `buscar_paciente`

3. **Verificar agendamento existente**
   Tool: `busca_agendamento`

4. **Buscar horários disponíveis**
   Tool: `buscar_horarios`

5. **Sugerir horários**
   Sempre 3 opções dentro do range retornado

6. **Confirmação**
   Aguarda confirmação explícita do paciente

7. **Salvar agendamento**
   Tool: `salvar_agendamento`

8. **Encerramento**
   Retorna os dados do agendamento e finaliza

---

## 🛠️ Tools (Banco de Dados)

### 🔹 buscar_paciente

```sql
SELECT id, nome_paciente
FROM paciente_dados_sensiveis
WHERE nome_paciente ILIKE '%' || :nome || '%'
LIMIT 1;
```

---

### 🔹 busca_agendamento

```sql
SELECT 
  a.id,
  a.horario_inicio,
  a.horario_fim,
  e.nome_estabelecimento
FROM agendamento_criado a
JOIN estabelecimento e ON e.id = a.id_estabelecimento
WHERE e.id_paciente = :id_paciente
ORDER BY a.horario_inicio DESC
LIMIT 1;
```

---

### 🔹 buscar_horarios

```sql
SELECT 
  e.id AS id_estabelecimento,
  e.nome_estabelecimento,
  e.range_horario
FROM estabelecimento e
WHERE e.id_paciente = :id_paciente;
```

---

### 🔹 salvar_agendamento

```sql
INSERT INTO agendamento_criado (
  id_paciente,
  id_estabelecimento,
  horario_inicio,
  horario_fim,
  created_at
)
VALUES (
  :id_paciente,
  :id_estabelecimento,
  :horario_inicio,
  :horario_fim,
  NOW()
)
RETURNING id, horario_inicio, horario_fim;
```

📌 Regra:

* `horario_fim = horario_inicio + 30 minutos`

---

## ⚠️ Regras Críticas do Agente

* ❌ Nunca pular etapas do fluxo

* ❌ Nunca inventar dados (paciente, horário, local)

* ❌ Nunca sugerir horários fora do range

* ❌ Nunca salvar sem confirmação explícita

* ✅ Sempre validar antes de salvar

* ✅ Sempre usar dados vindos das tools

* ✅ Sempre responder em JSON válido

---

## 📤 Envio das Respostas

### 🔹 Mensagens do Agente

* Faz o parse do JSON retornado pelo agente

### 🔹 Split Out

* Divide o array de mensagens em itens individuais

### 🔹 Delay (1s)

* Adiciona um pequeno atraso para simular conversa natural

### 🔹 Envia Mensagens (Telegram)

* Envia cada mensagem separadamente para o usuário

---

## 🚨 Tratamento de Erros

### 🔹 Mensagem erro

```text
Ops! Ocorreu um erro inesperado. Por favor, tente novamente em instantes.
```

---

## 🧩 Pontos Fortes do Workflow

* ✔ Fluxo conversacional bem controlado
* ✔ Uso de IA com regras rígidas (reduz erros)
* ✔ Suporte a áudio (melhor experiência do usuário)
* ✔ Integração direta com banco de dados
* ✔ Arquitetura modular com tools

---

## ⚠️ Possíveis Melhorias

* Cancelamento de agendamento
* Remarcação de consultas
* Validação de conflitos de horário
* Timeout de sessão
* Logs estruturados
* Monitoramento de erros

---

## 📎 Conclusão

Este workflow implementa um **assistente completo de agendamento automatizado**, combinando:

* n8n (orquestração)
* OpenAI (inteligência artificial)
* PostgreSQL (persistência de dados)
* Telegram (interface com o usuário)

Com um fluxo bem definido e controlado, o sistema garante consistência, reduz falhas e oferece uma experiência eficiente para o paciente.
