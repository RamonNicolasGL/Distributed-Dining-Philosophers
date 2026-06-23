# ProgConcorrenteTP2

# O Jantar dos Filósofos Distribuído em Elixir

Este projeto apresenta uma implementação do clássico problema de concorrência do **Jantar dos Filósofos** utilizando a linguagem **Elixir**. O objetivo deste código é servir como estudo de caso para soluções distribuídas baseadas em servidores replicados/independentes (`replicated servers`) e troca de mensagens.

A implementação transpõe rigorosamente um modelo de concorrência tradicional baseado em processos e operações bloqueantes para o **Modelo de Atores** nativo da máquina virtual do Erlang (BEAM).

---

## 📌 Contextualização do Problema

O Jantar dos Filósofos ilustra os desafios da alocação de recursos compartilhados em sistemas concorrentes. Cinco filósofos sentam-se à mesa para pensar e comer. Para comer, cada filósofo precisa de dois garfos (o da sua esquerda e o da sua direita). No entanto, existem apenas 5 garfos disponíveis.

### Desafios Resolvidos:
1. **Exclusão Mútua:** Dois filósofos vizinhos não podem comer ao mesmo tempo porque compartilham o mesmo garfo (gerenciado por um processo `Waiter`).
2. **Prevenção de Deadlock:** Se todos os filósofos pegassem o garfo esquerdo simultaneamente, o sistema travaria. A solução implementa uma **estratégia assimétrica** no quinto filósofo (`id = 4`), que inverte a ordem de requisição dos garfos, quebrando o ciclo de dependência.

---

## 🏗️ Arquitetura e Paradigma em Elixir

A solução utiliza o **Modelo de Atores**, onde cada entidade independente é um processo isolado que se comunica exclusivamente por troca de mensagens assíncronas ou síncronas.

### Componentes:

* **`Waiter` (Garçom / Servidor Replicado):** Implementado como um `GenServer`. Existem 5 instâncias independentes rodando em paralelo. Cada `Waiter` gerencia o estado de disponibilidade de um garfo específico (`:available` ou `:busy`). Ele gerencia nativamente a fila de filósofos que estão aguardando o recurso ser liberado.
* **`Philosopher` (Filósofo / Cliente Concorrente):** Implementado como um processo autônomo através de `spawn_link`. Ele executa um loop infinito (via recursão de cauda) alternando entre os estados de *Pensar*, *Requisitar Recursos (Bloqueante)*, *Comer* e *Liberar Recursos (Assíncrono)*.

---

## 🧬 Mapeamento do Código Original para Elixir

A tabela abaixo demonstra como o paradigma de comunicação do código original foi mapeado para as construções nativas do Elixir:

| Conceito Original | Implementação em Elixir | Explicação Técnica |
| :--- | :--- | :--- |
| `module Waiter[5]` | `Enum.each(0..4, ...)` | Criação de 5 processos `GenServer` independentes na memória. |
| `receive getforks()` | `handle_call(:get_forks, ...)` | Requisição síncrona. Se o garfo estiver ocupado, o servidor retém a resposta (`{:noreply, ...}`), bloqueando o cliente. |
| `receive relforks()` | `handle_cast(:rel_forks, ...)` | Mensagem assíncrona. O filósofo devolve o garfo e continua sua execução imediatamente. |
| `call Waiter[i].getforks()` | `GenServer.call({:global, ...})` | Chamada síncrona que pausa o processo do filósofo até obter o recurso. |
| `send Waiter[i].relforks()` | `GenServer.cast({:global, ...})` | Envio assíncrono de mensagem para o servidor. |
| `while (true)` | Encadeamento Recursivo (`p_loop/1`) | Loops infinitos em Elixir são otimizados pela BEAM através de funções que chamam a si mesmas no final (Tail Call Optimization), garantindo consumo zero de pilha de memória adicional. |

### O Fator Distribuído (`{:global, ...}`)
Os servidores `Waiter` são registrados utilizando a tupla `{:global, {:waiter, id}}`. Isso significa que o catálogo de processos é gerenciado globalmente pela rede da Erlang VM. Se este código for executado em um cluster de máquinas conectadas na mesma rede física, os filósofos de um nó conseguirão localizar e bloquear os garçons localizados em outros nós de maneira totalmente transparente.

---

## 🚀 Como Executar o Projeto

### Pré-requisitos
* Ter o [Elixir](https://elixir-lang.org/install.html) instalado na sua máquina (versão 1.12 ou superior recomendada).

### Execução como Script
1. Salve o código em um arquivo chamado `jantar.exs`.
2. Abra o terminal na pasta do arquivo e execute:
   ```bash
   elixir jantar.exs
