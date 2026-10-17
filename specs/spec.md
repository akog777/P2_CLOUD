# Especificação do Projeto: Sistema de Gestão para Clínica Veterinária (VetCare)

---

## 1. Problema

A clínica veterinária **VetCare** mantém os dados de tutores, pets e consultas espalhados em planilhas e anotações em papel. Isso causa três problemas no dia a dia:

1. **Recepção lenta:** para atender um cliente, o recepcionista precisa procurar o tutor e o pet em mais de um lugar, e cadastros duplicados são comuns.
2. **Agenda sem controle:** não há como saber, de forma confiável, se um veterinário já tem consulta marcada em determinado horário, o que gera conflitos de agenda.
3. **Falta de informação para a gestão:** a veterinária chefe não consegue ver o histórico de atendimentos de um animal durante a consulta nem acompanhar quantas consultas cada veterinário realiza, por espécie.

O VetCare é uma aplicação web, publicada na AWS, que centraliza tutores, pets e consultas em um único banco de dados, permitindo encontrar um cliente em poucos passos, agendar sem conflito de horário, consultar o histórico de cada pet, anexar fotos dos animais e emitir um relatório de atendimentos.

---

## 2. Personas

### Persona 1: Dra. Camila Rocha (Veterinária Chefe)
![Dra. Camila Rocha](persona-camila.jfif)
* **Perfil:** 38 anos, médica veterinária e sócia da clínica.
* **Necessidades:** ver rapidamente os dados, a foto e o histórico de consultas do pet durante o atendimento; acompanhar quantas consultas cada veterinário realiza, por espécie.
* **Dores:** perde tempo procurando o histórico do animal em anotações antigas e não tem nenhum relatório consolidado da clínica.

### Persona 2: Lucas Mendes (Recepcionista)
![Lucas Mendes](persona-lucas.jfif)
* **Perfil:** 24 anos, responsável pela recepção, pelos cadastros e pela agenda.
* **Necessidades:** encontrar um tutor pelo CPF ou nome, cadastrar e atualizar tutores e pets rapidamente e marcar consultas sabendo que o horário está livre.
* **Dores:** formulários longos, cadastros duplicados e conflitos de agenda descobertos só quando o cliente chega.

---

## 3. Como os Requisitos se Aplicam ao Domínio

### 3.1 Requisitos funcionais mínimos do enunciado

| ID | Requisito do enunciado | Aplicação no VetCare | Histórias |
| :--- | :--- | :--- | :--- |
| **RF-01** | CRUD de 2 entidades relacionadas no RDS | CRUD completo de **Tutores** e **Pets** (1 tutor : N pets). Cada pet também é vinculado a um **veterinário responsável**; os veterinários são cadastrados por carga inicial no banco e apenas listados no sistema | US-01, US-02 |
| **RF-02** | Upload de arquivo para o S3 via URL pré-assinada | **Foto do pet** enviada pelo navegador diretamente para o S3, usando uma URL pré-assinada gerada pelo backend | US-04 |
| **RF-03** | Evento S3 → Lambda → RDS | Cada nova foto dispara uma Lambda que gera a **miniatura** da imagem e grava o endereço dela no cadastro do pet | US-05 |
| **RF-04** | Relatório via API Gateway + Lambda com JOIN/GROUP BY | **Consultas realizadas por veterinário e espécie** em um período | US-08 |
| **RF-05** | Front-end web publicado consumindo a API | Interface web com as telas de tutores, pets, consultas e relatório | US-09 |

### 3.2 Requisitos funcionais específicos do domínio

| ID | Requisito | Aplicação no VetCare | Histórias |
| :--- | :--- | :--- | :--- |
| **RF-06** | Consultas | Agendar, remarcar, cancelar e registrar o atendimento de consultas (pet + veterinário + data/hora + motivo + observações + status), sem conflito de horário. Cria uma relação **N:N entre pets e veterinários** por meio das consultas | US-06 |
| **RF-07** | Histórico do pet | Listar todas as consultas de um pet, da mais recente para a mais antiga | US-07 |
| **RF-08** | Busca e filtros | Buscar tutores por nome ou CPF e filtrar pets por espécie, tutor e situação | US-03 |
| **RF-09** | Inativação de pet | Marcar um pet como inativo (óbito ou transferência) sem apagar seus dados e seu histórico, com possibilidade de reativação | US-02 |

### 3.3 Requisitos não funcionais

| ID | Requisito | Aplicação no VetCare | Histórias |
| :--- | :--- | :--- | :--- |
| **RNF-01** | RDS em subnet privada, sem acesso público, com Security Groups documentados | Os dados de tutores, pets e consultas ficam em um banco de dados **sem acesso pela internet**, acessível apenas pelo backend do sistema | US-01, US-02 |
| **RNF-02** | Nenhuma senha ou chave no repositório | Nenhuma senha do banco ou chave de acesso da AWS é armazenada no repositório do projeto | Todas |
| **RNF-03** | Funcionar numa nova sessão do Learner Lab sem IP fixo | O sistema volta a funcionar numa nova sessão do Learner Lab seguindo apenas o `README.md`, sem endereços IP fixos no código | US-09 |
| **RNF-04** | Decisão justificada (ADR): CRUD em Lambda ou container | A forma de execução do CRUD de tutores e pets é escolhida e justificada em um ADR no `design.md` | US-01, US-02 |
| **RNF-05** | Sem IaC (CloudFormation/Terraform/SAM) | Todos os recursos da AWS são criados pelo console ou pela CLI e documentados passo a passo no `README.md` | Todas |
| **RNF-06** | Proteção de dados pessoais (LGPD) — específico do domínio | CPF e telefone dos tutores são dados pessoais: nas listagens o CPF aparece mascarado e esses dados não são registrados em logs | US-01 |

---

## 4. Histórias de Usuário e Critérios de Aceite

Os critérios seguem o formato **Dado / Quando / Então**, para que cada um possa ser testado diretamente na API ou na interface. Toda mensagem de erro é exibida ao usuário em linguagem clara.

### US-01: Gerenciar tutores
**Requisito:** RF-01

* **Como** Lucas (recepcionista),
* **Quero** cadastrar, consultar, editar e excluir tutores (nome, CPF, telefone e e-mail),
* **Para** manter a base de clientes da clínica correta e sem duplicidade.

**Critérios de aceite:**
1. **Dado** um nome com 3 a 100 caracteres, um CPF válido e um telefone com 10 ou 11 dígitos, **quando** salvo o tutor, **então** ele é gravado no banco e a API retorna **201** com o registro criado.
2. **Dado** um nome vazio, um telefone fora do formato ou um CPF inválido (dígitos verificadores incorretos), **quando** tento salvar, **então** a API retorna **400** informando qual campo está incorreto e nada é gravado.
3. **Dado** um e-mail informado com formato inválido, **quando** tento salvar, **então** a API retorna **400**. O e-mail é opcional.
4. **Dado** um CPF que já pertence a outro tutor, **quando** tento cadastrar **ou editar**, **então** a API retorna **409** com a mensagem "CPF já cadastrado".
5. **Dado** o mesmo CPF digitado com e sem pontuação (`123.456.789-09` e `12345678909`), **quando** comparo, **então** o sistema trata os dois como o mesmo CPF.
6. **Dado** um tutor existente, **quando** edito seus dados, **então** as alterações ficam gravadas e aparecem na próxima consulta.
7. **Dado** a listagem de tutores, **quando** ela é exibida, **então** os tutores aparecem em ordem alfabética e o CPF é mostrado **mascarado** (`***.456.789-**`), por se tratar de dado pessoal. O CPF completo aparece somente na tela de detalhe/edição.
8. **Dado** o botão de excluir, **quando** clico nele, **então** o sistema pede confirmação antes de excluir.
9. **Dado** um tutor **com pets cadastrados** (ativos ou inativos), **quando** tento excluí-lo, **então** a API retorna **409** com a mensagem "Tutor possui pets cadastrados" e nada é removido.
10. **Dado** um tutor **sem pets**, **quando** confirmo a exclusão, **então** ele é removido e a API retorna **204**.
11. **Dado** um ID de tutor inexistente, **quando** consulto, edito ou excluo, **então** a API retorna **404**.

### US-02: Gerenciar pets
**Requisito:** RF-01

* **Como** Lucas (recepcionista),
* **Quero** cadastrar, consultar, editar, inativar e excluir os pets de um tutor, escolhendo o veterinário responsável,
* **Para** manter organizado o cadastro de pacientes de cada cliente.

**Critérios de aceite:**
1. **Dado** um tutor existente, **quando** cadastro um pet com nome, espécie (Cão, Gato, Ave, Roedor, Réptil ou Outro) e veterinário responsável, **então** o pet é gravado vinculado a esse tutor e a API retorna **201**.
2. **Dado** um tutor ou veterinário inexistente, **quando** tento salvar o pet, **então** a API retorna **400** informando qual vínculo é inválido.
3. **Dado** uma data de nascimento futura ou um peso menor ou igual a zero, **quando** tento salvar, **então** a API retorna **400**. Raça, data de nascimento e peso são opcionais.
4. **Dado** o formulário de pet, **quando** ele é aberto, **então** a lista de veterinários é carregada do sistema, não de valores fixos na tela.
5. **Dado** um pet existente, **quando** edito seus dados (inclusive trocando o veterinário responsável), **então** as alterações ficam gravadas.
6. **Dado** um tutor, **quando** consulto seus pets, **então** vejo todos os pets dele, com os inativos identificados por um selo "Inativo".
7. **Dado** a listagem geral de pets, **quando** nenhum filtro é aplicado, **então** apenas os pets ativos são exibidos.
8. **Dado** um pet ativo, **quando** o marco como inativo, **então** ele continua cadastrado como inativo e seu histórico de consultas é mantido.
9. **Dado** um pet com consultas **agendadas no futuro**, **quando** o inativo, **então** essas consultas são **canceladas automaticamente** e o sistema informa quantas foram canceladas.
10. **Dado** um pet inativo, **quando** o reativo, **então** ele volta a aparecer na listagem geral e pode ter novas consultas agendadas.
11. **Dado** o botão de excluir, **quando** clico nele, **então** o sistema pede confirmação antes de excluir.
12. **Dado** um pet **sem consultas**, **quando** confirmo a exclusão, **então** ele é removido e a API retorna **204**.
13. **Dado** um pet **com consultas**, **quando** tento excluí-lo, **então** a API retorna **409** com a mensagem "Pet possui consultas; utilize a inativação".
14. **Dado** um ID de pet inexistente, **quando** consulto, edito ou excluo, **então** a API retorna **404**.

### US-03: Buscar tutores e pets
**Requisito:** específico do domínio

* **Como** Lucas (recepcionista),
* **Quero** encontrar um tutor pelo nome ou CPF e filtrar pets por espécie, tutor ou situação,
* **Para** atender o cliente no balcão sem procurar em listas longas.

**Critérios de aceite:**
1. **Dado** um trecho do nome (ex.: "silv"), **quando** busco tutores, **então** recebo os tutores cujo nome contém o trecho, sem diferenciar maiúsculas, minúsculas e acentos.
2. **Dado** um CPF completo, com ou sem pontuação, **quando** busco, **então** recebo o tutor correspondente.
3. **Dado** uma busca sem resultados, **quando** ela é feita, **então** a tela mostra "Nenhum tutor encontrado".
4. **Dado** um campo de busca vazio, **quando** busco, **então** recebo a listagem normal.
5. **Dado** filtros de espécie, tutor e/ou situação (ativo/inativo), **quando** listo pets, **então** recebo apenas os pets que atendem a todos os filtros informados.
6. **Dado** mais de 20 resultados, **quando** listo tutores ou pets, **então** recebo os resultados em páginas de 20 itens, com o total de registros e a navegação entre páginas.

### US-04: Enviar a foto do pet
**Requisito:** RF-02

* **Como** Lucas (recepcionista),
* **Quero** anexar a foto do pet no cadastro dele,
* **Para** que a equipe reconheça o animal facilmente durante o atendimento.

**Critérios de aceite:**
1. **Dado** um pet existente e um arquivo JPG ou PNG de até 5 MB, **quando** solicito o envio, **então** o backend retorna uma **URL pré-assinada válida por 5 minutos**.
2. **Dado** a URL pré-assinada, **quando** a foto é enviada, **então** o arquivo vai **direto do navegador para o S3**, sem passar pelo backend.
3. **Dado** um arquivo com outro formato ou com mais de 5 MB, **quando** tento enviar, **então** a tela bloqueia o envio com uma mensagem, e o backend recusa gerar a URL para formatos não permitidos (**400**).
4. **Dado** uma URL expirada, **quando** o envio é tentado, **então** o envio é recusado e a tela orienta o usuário a tentar novamente.
5. **Dado** um pet que já tinha foto, **quando** envio uma nova, **então** a nova foto passa a ser a foto do pet.
6. **Dado** um pet sem foto, **quando** ele é exibido, **então** aparece uma imagem padrão.

### US-05: Gerar a miniatura da foto automaticamente
**Requisito:** RF-03

* **Como** Dra. Camila (veterinária chefe),
* **Quero** que as listagens mostrem miniaturas leves das fotos dos pets,
* **Para** que as telas carreguem rápido mesmo com muitos pacientes.

**Critérios de aceite:**
1. **Dado** uma nova foto enviada para o S3, **quando** o arquivo é criado, **então** uma Lambda é disparada automaticamente.
2. **Dado** o processamento, **quando** a Lambda termina, **então** a miniatura (150x150 px, mantendo a proporção) é salva no S3 e os endereços da foto e da miniatura são gravados no cadastro do pet.
3. **Dado** a miniatura recém-salva no S3, **quando** ela é criada, **então** ela **não** dispara um novo processamento, evitando execuções em loop.
4. **Dado** que a foto acabou de ser enviada e a miniatura ainda não está pronta, **quando** o pet é exibido, **então** aparece o indicador "processando".
5. **Dado** um arquivo corrompido ou que não pertence a um pet existente, **quando** a Lambda o processa, **então** o erro é registrado no log e nenhum cadastro é alterado.
6. **Dado** qualquer processamento, **quando** ele ocorre, **então** o log do CloudWatch registra o início, o fim e o resultado.

### US-06: Agendar e registrar consultas
**Requisito:** específico do domínio

* **Como** Lucas (recepcionista) e Dra. Camila (veterinária),
* **Quero** agendar consultas para os pets e, depois do atendimento, registrar o resultado,
* **Para** organizar a agenda dos veterinários e manter o histórico dos pacientes.

**Regras da agenda:** cada consulta dura **30 minutos**; a clínica atende de **segunda a sábado, das 8h às 18h**; a consulta pode ser feita com um veterinário diferente do responsável pelo pet (plantão ou especialista).

**Critérios de aceite:**
1. **Dado** um pet ativo, um veterinário, uma data/hora futura dentro do horário de funcionamento e um motivo, **quando** agendo, **então** a consulta é criada com status `agendada` e a API retorna **201**.
2. **Dado** que o veterinário já tem uma consulta não cancelada que se sobrepõe ao intervalo de 30 minutos, **quando** tento agendar, **então** a API retorna **409** com a mensagem "Veterinário já possui consulta neste horário".
3. **Dado** que o **pet** já tem uma consulta não cancelada que se sobrepõe ao horário, mesmo com outro veterinário, **quando** tento agendar, **então** a API retorna **409** com a mensagem "Pet já possui consulta neste horário".
4. **Dado** uma consulta das 10:00 às 10:30, **quando** agendo outra para o mesmo veterinário às 10:30, **então** o agendamento é aceito, pois os horários apenas se encostam.
5. **Dado** uma data/hora passada, um horário fora do funcionamento da clínica ou um pet inativo, **quando** tento agendar, **então** a API retorna **400** informando o motivo.
6. **Dado** dois agendamentos enviados ao mesmo tempo para o mesmo veterinário e horário, **quando** são processados, **então** apenas um é aceito e o outro recebe **409**.
7. **Dado** uma consulta `agendada`, **quando** a remarco, **então** a nova data/hora é validada pelos critérios 2 a 5, **sem** considerar conflito com a própria consulta (ex.: mudar das 10:00 para as 10:15 é permitido se não houver outra consulta no intervalo).
8. **Dado** uma consulta `agendada` cujo horário já chegou, **quando** a Dra. Camila registra o atendimento com observações, **então** o status passa para `realizada`.
9. **Dado** uma consulta `agendada` cujo horário **ainda não chegou**, **quando** tento registrá-la como `realizada`, **então** a API retorna **400**.
10. **Dado** uma consulta `agendada`, **quando** ela é cancelada, **então** o status passa para `cancelada` e o horário fica livre para outro agendamento.
11. **Dado** uma consulta `realizada` ou `cancelada`, **quando** tento mudar seu status ou remarcá-la, **então** a API retorna **409**.
12. **Dado** uma data e um veterinário, **quando** abro a agenda, **então** vejo as consultas daquele dia em ordem de horário.

### US-07: Consultar o histórico do pet
**Requisito:** específico do domínio

* **Como** Dra. Camila (veterinária chefe),
* **Quero** ver todas as consultas de um pet durante o atendimento,
* **Para** conhecer o histórico do animal sem procurar em anotações antigas.

**Critérios de aceite:**
1. **Dado** um pet com consultas, **quando** abro o histórico, **então** vejo as consultas da mais recente para a mais antiga, com data, veterinário, motivo, status e observações.
2. **Dado** o histórico aberto, **quando** filtro por status, **então** vejo apenas as consultas daquele status (ex.: somente realizadas).
3. **Dado** um pet sem consultas, **quando** abro o histórico, **então** vejo a mensagem "Nenhuma consulta registrada".
4. **Dado** um pet inativo, **quando** abro o histórico, **então** o histórico continua disponível.

### US-08: Emitir o relatório de consultas por veterinário e espécie
**Requisito:** RF-04

* **Como** Dra. Camila (veterinária chefe),
* **Quero** ver quantas consultas cada veterinário realizou, separadas pela espécie do animal, em um período,
* **Para** acompanhar a distribuição da demanda e planejar a escala da equipe.

**Critérios de aceite:**
1. **Dado** um período (data inicial e final), **quando** solicito o relatório, **então** recebo, para cada veterinário e espécie, a quantidade de consultas com status `realizada` no período.
2. **Dado** que nenhum período é informado, **quando** solicito, **então** o relatório considera os últimos 30 dias.
3. **Dado** uma data inicial maior que a final, **quando** solicito, **então** a API retorna **400**.
4. **Dado** um período sem consultas realizadas, **quando** solicito, **então** recebo um resultado vazio com status **200** e a tela mostra "Nenhuma consulta no período".
5. **Dado** a implementação, **então** o relatório é exposto pelo API Gateway, executado por uma Lambda e usa uma consulta SQL com `JOIN` entre consultas, pets e veterinários e `GROUP BY` por veterinário e espécie.
6. **Dado** a resposta, **então** ela segue o formato JSON:
   ```json
   {
     "periodo": { "inicio": "2026-10-01", "fim": "2026-10-31" },
     "resultado": [
       { "veterinario": "Dra. Camila Rocha", "especie": "Cão",  "total_consultas": 12 },
       { "veterinario": "Dra. Camila Rocha", "especie": "Gato", "total_consultas": 5 }
     ]
   }
   ```

### US-09: Usar o sistema pelo navegador
**Requisito:** RF-05

* **Como** Lucas (recepcionista),
* **Quero** acessar o sistema pelo navegador, sem instalar nada,
* **Para** usar qualquer computador da recepção.

**Critérios de aceite:**
1. **Dado** o endereço público do site, **quando** o acesso, **então** vejo as telas de tutores, pets, consultas e relatório.
2. **Dado** uma operação em andamento, **quando** aguardo a resposta, **então** a tela mostra um indicador de carregamento.
3. **Dado** um erro retornado pela API, **quando** ele ocorre, **então** a tela mostra a mensagem de erro, e não uma tela em branco.
4. **Dado** que a API está indisponível, **quando** tento usar o sistema, **então** a tela mostra "Não foi possível conectar ao servidor. Tente novamente."
5. **Dado** uma nova sessão do Learner Lab, **quando** a equipe segue o `README.md`, **então** o site volta a funcionar sem alterar endereços dentro do código das telas.
