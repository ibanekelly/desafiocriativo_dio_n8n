Atue como um especialista em N8N.



Crie uma automação para enviar lembretes de agendamento por WhatsApp e reduzir faltas em atendimentos marcados.



**Público:**
Prestadores de serviço e pequenos negócios que trabalham com agenda, como a equipe de atendimento ou o próprio dono do negócio.



**Ferramentas envolvidas:**
n8n, Google Sheets e uzapi (API de WhatsApp).



**Fluxo:**

1. Todos os dias, em um horário fixo (por exemplo, 9h), o n8n inicia o fluxo automaticamente.
2. Ler a planilha de agendamentos no Google Sheets (ID único do agendamento, nome do cliente, telefone, data, horário e status do lembrete).
3. Filtrar apenas os atendimentos marcados para o dia seguinte e que ainda não receberam lembrete.
4. Para cada atendimento, montar uma mensagem personalizada com o nome do cliente, a data e o horário.
5. Enviar a mensagem pelo WhatsApp via uzapi.
6. Atualizar a planilha, marcando o status do lembrete como "enviado" e registrando o horário do envio.



**Regras:**

* Enviar o lembrete somente uma vez por agendamento (ignorar linhas com status "enviado").
* Ignorar linhas sem telefone ou com telefone inválido e marcar o status como "erro" para revisão manual.
* Padronizar o telefone para o formato internacional, só com dígitos (código do país + DDD + número).
* Usar mensagem de template aprovada quando o cliente não tiver interagido nas últimas 24 horas.
* Colocar um intervalo de alguns segundos entre os envios para evitar bloqueio do número.
* Não enviar lembrete para atendimentos cancelados.
* Se o envio falhar, registrar o erro na planilha e não marcar como "enviado".
* Durante os testes, o canal de envio pode ser substituído por Telegram (gratuito), mantendo o restante do fluxo igual.



**Nós utilizados:**

1 - **Schedule Trigger** - Inicia o workflow todos os dias em horário fixo (ex.: 9h).

2 - **Google Sheets (Get Row(s))** - Lê todas as linhas da planilha de agendamentos.

3 - **Filter** - Mantém só os agendamentos de amanhã, com telefone preenchido, sem status "enviado" e sem status "cancelado".

4 - **Edit Fields (Set)** ou **Code** - Padroniza o telefone (só dígitos, com 55 + DDD) e monta o texto da mensagem com nome, data e horário.

5 - **Loop Over Items** - Processa um agendamento por vez, para controlar o ritmo dos envios.

6 - **HTTP Request** - Faz o POST na API da uzapi para enviar a mensagem de WhatsApp (autenticação por Header Auth com Bearer token).

7 - **Wait** - Aguarda alguns segundos entre um envio e outro, para evitar bloqueio do número.

8 - **Google Sheets (Update Row)** - Em caso de sucesso: grava status "enviado" e o horário do envio na linha do agendamento.

9 - **Google Sheets (Update Row)**, ramo de erro - Em caso de falha (HTTP Request com "On Error: Continue (using error output)"): grava status "erro" e a mensagem de erro.



**Lógica do workflow:**

1. **Disparo diário.** O Schedule Trigger roda sozinho no horário definido, sem ação manual.
2. **Leitura.** O Google Sheets traz todas as linhas. Cada agendamento tem um ID único, usado depois para atualizar a linha certa.
3. **Seleção.** O Filter compara a data de cada linha com a data de amanhã (no mesmo formato da planilha) e descarta linhas já enviadas, canceladas ou sem telefone. Linhas sem telefone ou com telefone inválido podem seguir por um ramo separado que grava "erro" para revisão manual.
4. **Preparação.** O nó Edit Fields limpa o telefone (remove espaços, traços e parênteses, adiciona o código do país) e cria o campo da mensagem, por exemplo: "Olá, \[nome]! Lembrando do seu atendimento amanhã, \[data], às \[horário]. Pode confirmar presença?".
5. **Envio item a item.** O Loop Over Items entrega um agendamento por vez ao HTTP Request, que chama a uzapi com o telefone e o texto. Para clientes fora da janela de 24 horas, o corpo da requisição usa uma mensagem de template aprovada em vez de texto livre.
6. **Intervalo.** O nó Wait pausa por alguns segundos antes de voltar ao loop, espaçando os envios.
7. **Registro do resultado.**

   * Se o envio funcionou, o Update Row marca "enviado" e registra o horário. Isso impede mensagens duplicadas, porque o Filter ignora linhas com esse status nas próximas execuções.
   * Se o envio falhou, o ramo de erro marca "erro" e guarda o motivo, e a linha não fica como "enviado". Assim ela pode ser revisada ou reenviada.
8. **Fim do ciclo.** Quando o loop termina todos os itens, o workflow encerra e aguarda o próximo disparo do dia seguinte.



