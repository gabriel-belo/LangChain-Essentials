# LangChain-Essentials
Curso da LangChain Academy "LangChain Essemtials - Python"


Injeção de Dependência (DI) é um padrão de design que permite que um objeto (sua ferramenta execute_sql) receba as coisas das quais ele precisa (a conexão com o banco de dados db) de uma fonte externa (o LangGraph), em vez de criá-las por conta própria.

Sem Injeção: Sua função teria que criar o db (conexão) toda vez que fosse chamada, o que é ineficiente:

~~~Python

def execute_sql(query: str) -> str:
    db = SQLDatabase.from_uri("sqlite:///Chinook.db") # Ruim!
    # ...
~~~

Com Injeção (O que o LangGraph faz): A conexão com o banco (db) é criada uma única vez no início, e depois é passada (ou injetada) para todas as funções que precisam dela, através do RunTimeContext.

"Injetado" significa que o LangGraph pega o valor real (SQLDatabase.from_uri(...)) e o insere no contexto de execução.

## Runtime Context
O Runtime Context no LangChain (e mais especificamente no LangGraph) é o mecanismo usado para gerenciar e injetar dependências estáticas em seus agentes e ferramentas. Ele atua como um contêiner de informações que são necessárias para a execução, mas que não devem ser alteradas durante o run (execução) do agente, e frequentemente não devem ser expostas ao LLM.

É um padrão de Injeção de Dependência para aplicações de LLM.

### Componentes do Runtime
O LangGraph expõe um objeto Runtime que contém informações cruciais para a execução do agente. O Runtime Context é a parte desse objeto que lida com dados estáticos.

Context: 
* Tipo de Informação: Estático e imutável.
* Propósito: Configuração e dependências que as ferramentas precisam. 	
* Exemplo: Conexão com Banco de Dados (SQLDatabase), ID do Usuário, API Keys, configurações do ambiente.


Store: 	
* Tipo de Informação: Dinâmico e mutável.	
* Propósito: Memória de longo prazo que persiste entre diferentes runs ou conversas.	
* Exemplo: Perfil do usuário, histórico de interações salvas em um banco de dados.


Stream Writer:	
* Tipo de Informação: Utilitário.	
* Propósito: Objeto usado para transmitir (streaming) informações de progresso em tempo real.	
* Exemplo: Atualizações de status da ferramenta (ex: "Buscando dados...", "Calculando...").

### 💉 Injeção de Dependência em Ação
O principal papel do Runtime Context é evitar o uso de variáveis globais e garantir que seus componentes sejam testáveis e reutilizáveis.

1. Definição do Esquema
Você primeiro define a estrutura do seu contexto usando um dataclass ou context_schema, como você viu:
~~~ Python

from dataclasses import dataclass
@dataclass
class RunTimeContext:
    db: SQLDatabase # Definindo que o contexto precisa de um objeto SQLDatabase
    user_id: str    # Definindo que o contexto precisa do ID do usuário
~~~

2. A Injeção na Invocação
A instância real dessas dependências é fornecida quando o agente ou o LangGraph é invocado (iniciado):

~~~Python

db_conexao = SQLDatabase.from_uri("sqlite:///Chinook.db")
contexto_real = RunTimeContext(db=db_conexao, user_id="usuario_xyz")

# A injeção ocorre aqui, o LangGraph associa o contexto_real à execução:
agente.invoke(..., context=contexto_real)
~~~

3. Acesso Dentro das Ferramentas
Dentro de qualquer ferramenta (@tool) ou middleware, você acessa o objeto Runtime (geralmente usando get_runtime ou um argumento runtime) para buscar o recurso injetado:

~~~Python

@tool
def execute_query() -> str:
    runtime = get_runtime(RunTimeContext)
    # Acessa o objeto SQLDatabase que foi injetado:
    db = runtime.context.db 
    # Acessa o ID do usuário que foi injetado:
    user_id = runtime.context.user_id 
    # ... executa a lógica ...
~~~

O Runtime Context permite que as suas ferramentas (execute_query no exemplo) sejam "cegas" sobre como a conexão (db) foi criada, elas apenas sabem que ela está disponível no contexto.

### 🏗️ Especificando o context_schema
A especificação do context_schema é uma forma mais formal e robusta, usada pelo LangGraph (e pelo LangChain em ambientes complexos), de definir a estrutura exata do Runtime Context que você viu.

É essencialmente o contrato que garante que o motor de execução (o runtime) terá todos os recursos necessários antes que o agente comece a trabalhar.

#### O que é o context_schema?
O context_schema é a definição da classe Python (muitas vezes uma dataclass) que o LangGraph usa para saber quais dependências e configurações estáticas ele deve carregar no contexto de execução.

Ele tem duas funções principais:

* Declaração de Requisitos: Informa ao LangGraph quais objetos devem ser injetados no contexto.

* Validação: Permite que o LangGraph verifique, logo no início, se o contexto fornecido na invocação (agent.invoke(..., context=...)) realmente atende a esses requisitos.