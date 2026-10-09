<div align=center>

# UniCesumar

**Engenharia de Software**

<br>

**João Victor Marcondes da Silva - 25184341-2**<br>
**Thiago Emanuel de Matos Saboia - 25283904-2**


<br><br><br><br>

## ARQUITETURA E CONCORRÊNCIA
**Estudo de caso: O Caos do Show de Rock**

<br><br><br><br>

---

**Londrina**  
**2026**

</div>

<br><br>
  
## 1. INTRODUÇÃO

Um sistema de venda de ingressos para um show internacional precisa lidar com muitos acessos em um curto período. Neste estudo de caso, a previsão é de 100.000 acessos simultâneos no momento em que as vendas forem abertas. Além de suportar a quantidade de requisições, o sistema precisa impedir que duas pessoas comprem o mesmo assento.

Para compreender esse desafio, este relatório apresenta os conceitos de processo e thread, compara threads em modo usuário e em modo núcleo, explica a condição de corrida e descreve uma solução baseada em exclusão mútua e regiões críticas. Também é importante observar que threads, isoladamente, não garantem capacidade para atender 100.000 acessos: a aplicação precisa de uma arquitetura escalável, controle de carga e consistência no armazenamento dos ingressos.

## 2. SÍNTESE TEÓRICA

### 2.1 Diferença entre processo e thread

Um **processo** é um programa em execução, acompanhado dos recursos e das informações que o sistema operacional administra para ele. Em geral, possui seu próprio espaço de endereçamento virtual, além de recursos como arquivos abertos e informações de segurança.

Uma **thread** é uma unidade de execução dentro de um processo. Um mesmo processo pode ter várias threads executando tarefas diferentes. As threads de um processo compartilham o espaço de endereçamento e os recursos desse processo, mas cada thread possui sua própria pilha, seus registradores e seu contador de programa.

No sistema de ingressos, por exemplo, o serviço de aplicação pode utilizar threads para processar requisições de clientes, consultar informações e preparar respostas. Como as threads do mesmo processo compartilham memória, é necessário sincronizar o acesso aos dados compartilhados.

### 2.2 Por que threads costumam ser mais leves que processos?

Criar um processo normalmente exige preparar um novo contexto de execução e um espaço de endereçamento separado, além de administrar seus recursos. Já uma thread criada dentro de um processo existente pode reutilizar o espaço de endereçamento e os recursos desse processo. Por isso, sua criação e a troca de contexto entre threads do mesmo processo costumam ter menor custo do que operações equivalentes entre processos independentes.

O compartilhamento de memória também facilita a comunicação entre threads, pois elas podem acessar estruturas de dados comuns. Entretanto, esse benefício traz um risco: duas threads podem tentar alterar o mesmo dado ao mesmo tempo, causando inconsistência se não houver sincronização.

Essa diferença não significa que seja adequado criar uma thread para cada um dos 100.000 usuários. Uma quantidade tão grande de threads pode consumir muita memória e gerar custos de escalonamento. Em um sistema real, é comum utilizar um conjunto limitado de threads, filas de trabalho, operações assíncronas, controle de concorrência e múltiplas instâncias do serviço. O dimensionamento deve ser validado por testes de carga.

### 2.3 Threads em modo usuário e modo núcleo (kernel)

As **threads em modo usuário** são gerenciadas, em determinadas implementações, por uma biblioteca ou pelo próprio ambiente de execução da aplicação, sem que cada operação de gerenciamento precise envolver diretamente o núcleo do sistema operacional. Isso pode reduzir o custo de certas operações. Porém, dependendo do modelo utilizado, uma chamada bloqueante pode impedir o avanço de outras threads gerenciadas no mesmo contexto, e o sistema operacional pode não escalonar cada thread de usuário de forma independente.

As **threads em modo núcleo** são conhecidas e gerenciadas pelo sistema operacional. O núcleo pode escaloná-las individualmente e distribuí-las entre processadores disponíveis. Esse modelo oferece integração direta com o escalonador do sistema operacional, mas operações de gerenciamento podem ter maior custo.

É importante diferenciar o modo de execução da CPU do modelo de gerenciamento de threads: “modo usuário” e “modo núcleo” também descrevem níveis de privilégio do processador. Sistemas operacionais modernos podem combinar mecanismos diferentes para implementar concorrência; portanto, o comportamento exato depende da plataforma.

## 3. DIAGNÓSTICO DO PROBLEMA: CONDIÇÃO DE CORRIDA

### 3.1 O que é uma condição de corrida?

Uma **condição de corrida** (*race condition*) ocorre quando o resultado de uma operação depende da ordem ou do momento em que diferentes threads ou processos acessam e modificam um recurso compartilhado, sem uma coordenação adequada.

Considere que o assento A-15 esteja inicialmente disponível. O Usuário A e o Usuário B clicam em “Comprar” praticamente ao mesmo tempo. Se as duas requisições consultarem o estado do assento antes que qualquer uma registre a reserva, ambas podem concluir que ele está livre.

Uma sequência problemática seria:

1. A requisição do Usuário A consulta o assento A-15 e encontra o estado “disponível”.
2. A requisição do Usuário B consulta o mesmo assento antes da atualização e também encontra “disponível”.
3. As duas requisições tentam confirmar a compra.
4. Sem uma proteção adequada, o sistema pode registrar duas compras para o mesmo assento.

As consequências podem incluir venda duplicada, cobrança indevida, divergência entre o ingresso enviado ao cliente e o estado armazenado, além de perda de confiança no serviço. O problema não é resolvido apenas porque o código usa threads ou porque o banco de dados é rápido: é necessário garantir que a operação de reserva seja consistente.

## 4. SOLUÇÃO ARQUITETURAL

### 4.1 Exclusão mútua e regiões críticas

A **exclusão mútua** é uma técnica de sincronização que garante que apenas uma thread por vez acesse uma determinada região crítica, quando essa região exige acesso exclusivo a um recurso compartilhado.

Uma **região crítica** é o trecho do programa que consulta ou modifica um recurso compartilhado e que precisa ser protegido contra acessos concorrentes incompatíveis. No caso dos ingressos, a região crítica envolve verificar se o assento está disponível e realizar a reserva de forma indivisível.

Um mecanismo comum em programação concorrente é o *mutex* (bloqueio de exclusão mútua). A thread precisa adquirir o bloqueio antes de entrar na região crítica e liberá-lo ao terminar. Se outra thread tentar adquirir o mesmo bloqueio enquanto ele estiver ocupado, deverá aguardar.

### 4.2 Aplicação ao assento A-15

Em uma única instância da aplicação, um bloqueio associado ao assento poderia organizar o acesso ao recurso. A primeira requisição entraria na região crítica, verificaria a disponibilidade e reservaria o assento. A segunda aguardaria; ao entrar, encontraria o assento já reservado e receberia uma mensagem informando que ele não está mais disponível.

Entretanto, um mutex mantido apenas na memória de uma instância não é suficiente quando o sistema possui várias instâncias ou servidores. Para um serviço de venda real, a garantia precisa existir no nível compartilhado por todas as instâncias — normalmente no banco de dados ou em outro mecanismo distribuído confiável.

Uma estratégia é realizar a reserva em uma transação de banco de dados com uma operação atômica e uma restrição que impeça duas reservas confirmadas para o mesmo assento. Por exemplo, o sistema pode atualizar o assento somente se seu estado ainda for “disponível” e confirmar a compra apenas quando exatamente um registro tiver sido alterado. Uma restrição de unicidade sobre o evento e o assento também pode impedir reservas duplicadas. A transação deve tratar corretamente conflitos e falhas.

#### Exemplo simplificado de fluxo

1. Receber a solicitação de compra do assento A-15.
2. Iniciar uma transação.
3. Tentar alterar o assento de “disponível” para “reservado”, condicionando a alteração ao estado atual.
4. Se apenas uma solicitação conseguir realizar a alteração, concluir a reserva e confirmar a transação.
5. Se nenhuma linha for alterada porque o assento já foi reservado, cancelar a tentativa e informar ao cliente que o assento não está disponível.
6. Finalizar a transação e liberar os recursos utilizados.

Assim, mesmo que as requisições cheguem no mesmo milissegundo, apenas uma delas poderá confirmar a reserva. A outra receberá uma resposta de conflito. Em sistemas reais, também devem ser considerados tempo limite de reserva, expiração de carrinho, idempotência para evitar cobranças repetidas e recuperação de falhas.

### 4.3 Capacidade para 100.000 acessos simultâneos

A sincronização impede a duplicidade, mas não resolve sozinha o desafio de capacidade. Para suportar picos de acesso, a arquitetura deve ser dimensionada e testada. Algumas medidas relevantes são:

- utilizar balanceamento de carga e múltiplas instâncias da aplicação;
- limitar a quantidade de requisições simultâneas e aplicar filas ou sala de espera virtual;
- usar pools de threads e operações assíncronas conforme a tecnologia;
- monitorar CPU, memória, latência, erros e saturação do banco de dados;
- implementar transações, restrições de integridade e tratamento de conflitos;
- realizar testes de carga e de estresse antes da abertura das vendas.

A solução combina, portanto, sincronização correta dos dados com escalabilidade, observabilidade e testes. O objetivo não é apenas responder rapidamente, mas garantir que cada assento seja vendido no máximo uma vez.

## 5. CONSIDERAÇÕES FINAIS

O estudo demonstra que processos e threads são conceitos fundamentais para compreender como um sistema operacional executa tarefas concorrentes. Threads costumam ser mais leves do que processos porque compartilham recursos do processo, mas esse compartilhamento exige cuidado com dados acessados simultaneamente.

No caso do assento A-15, a condição de corrida pode permitir que duas requisições observem o mesmo estado disponível e tentem concluir a compra. A exclusão mútua e a proteção das regiões críticas ajudam a coordenar o acesso; em uma aplicação distribuída, essa garantia precisa ser reforçada por transações e restrições de integridade no armazenamento compartilhado.

Por fim, atender 100.000 acessos simultâneos exige mais do que criar threads: é necessário controlar a carga, distribuir requisições, proteger as operações de compra e validar a arquitetura por meio de testes.

## REFERÊNCIAS

MICROSOFT. **About processes and threads**. Microsoft Learn, [s. d.]. Disponível em: https://learn.microsoft.com/en-us/windows/win32/procthread/about-processes-and-threads. Acesso em: 8 out. 2026.

MICROSOFT. **User mode and kernel mode**. Microsoft Learn, [s. d.]. Disponível em: https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/user-mode-and-kernel-mode. Acesso em: 8 out. 2026.

SILBERSCHATZ, Abraham; GALVIN, Peter Baer; GAGNE, Greg. **Operating system concepts**. 10. ed. Hoboken: Wiley, 2018. Disponível em: https://shre.ink/Prof-Carlos-Maziero. Acesso em: 8 out. 2026.

MAZIERO, Carlos Alberto. Sistemas operacionais: conceitos e mecanismos. Curitiba: UFPR, 2019. Disponível em: https://shre.ink/h0Qv. Acesso em: 8 out. 2026.

---
