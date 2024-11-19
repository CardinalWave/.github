# CardinalWave: Um serviço de comunicação idependente

Este projeto contem um aplicação completa de desenvolvida para demonstrar a utilziação da rede CardinalCloud construida ao longo do projeto. Sua principal função e demonstrar a simplicidade e a utilização dos conceitos desenvolvidos ao longo do curso, atraves da disponibilização de um serviço de comunicação com funcionamento externo as redes convencionais.
Demonstrando o pleno funcionamento das integrações entre microcontroladores e usuaurios presentes na rede, alem de insentivar novas aplicações criadas pela comunidade de desenovolvimento.

O exemplo foi projetado para ser usado em um cenario em que a comunicacao via internet se encontra inutilizada em um determinada região, e á necessidade da comunicação entre os afetados.

O objetivo do programa é permitir que qualquer pessoa que tenha a disponibilidade algum meio de conexão a rede Wi-Fi, consiga se conectar a rede CardinalCloud, e acessar a aplicação criada, possibilitando um meio de contato na regiao local.

Como nosso objetivo com está aplicação é realizar a comunicação entre úsuario conectados á rede. Em nosso projeto contamos com operação de: 
    (1) gerenciamento de conta (login de usuario e registro), 
    (2) gerenciamento de grupos (criar e entrar) e 
    (3) envio de mensagens.

<p align="center">
    <img width="70%" src="https://user-images.githubusercontent.com/7620947/108773349-f68f3500-753c-11eb-8c4f-434ca9a9deec.png" />
</p>

No restante deste documento vamos:

-   Descrever o sistema, com foco na sua arquitetura.
-   Apresentar instruções para sua execução local, usando o código disponibilizado no repositório.
-   Operações implementas, e seu funcionamento entre os serviços.
-   Funcionamento dos containers Docker para facilitar a execução dos microsserviços.

## Arquitetura

CardinalWave possui seis microsserviços:

-   cw-bff-service: microsserviço responsável pela interface e o processo de Bridge (Ponte) entre os protocolos MQTT e WebSocket.
-   cw-mqtt-gateway: microsserviço responsável por intermediar a comunicação entre o broker Mosquitto-MQTT e o demais serviços.
-   cw-central-service: microserviço para centralização dos processos.
-   cw-auth-service: microserviço utilizado para a authenticação do usuario.
-   cw-log-trace: microserviço responsável pelo gerenciamento de logs gerados pelas demais aplicações.

Os seis microsserviços estão implementados em **Python**, usando o Flask para execução dos serviços no back-end.

Tambem contamos com a utilização de bancos de dados e serviços externos.
-   cw_central_db: banco de dados conectado a cw-central-service, sua principal função é gerenciar sessoes e grupos.

-   cw_message_db: banco de dados conectado a cw-message-service, sua principal função é gerenciar chats ativos e mensagens.

-   keycloak: serviço utilizado para autenticar e gerar o token do usuario.

-   Mosquitto-Broker (Broker): Broker utilizado para gerir os eventos publicados e entregar para os servicos inscritos nos topicos presentes.

## Protocolos de Comunicação

Como ilustrado no diagrama a seguir, a comunicação entre o cliente (cw-bff-service) e o backend usa um **BROKER**, como é comum no caso de sistemas que utilizam eventos para gerenciamento de suas ações.

Já a comunicação entre o cw-mqtt-gateway e os microsserviços do back-end é baseada em [REST](https://grpc.io/).

Definimos dois principais tipos de requisições gerais da aplicação afim de aproveitar a extensibilidade presente no protocolo MQTT e a utilização de REST por sua preseça na maioria dos frameworks Python.
Temos então, ações simples, aqueles que mantem o processamento da requisição apenas entre os serviços descritos na imagem.

<p align="center">
    <img width="70%" src="acao_simples.png" />
</p>
 - Ação Simples

#### Ação Simples
Nesta exemplificação do sistema temos a representação das ações consideradas simples, em que as requisições do usuario interagem apenas com os microserviços descritos, o todo o processamento ocorre em grande parte em **CW-CENTRAL-SERVICE**, e os componentes em que está conectado. Estes eventos são relacionados a interações individuais do úsuario sendo neste caso:
    
    - Login: Quando o usuario entra em sua conta.
    - Registrar: O úsuario cria sua conta.
    - Criar grupo: Requisição para a criação de um novo grupo
    - Entrar no Grupo: Requisição para a entrada em um grupo ja criado 
    - Entrar no Chat: Requisção para a entrada em um chat de grupo
    - Erros de requisição

Desta forma simplificamos o processo das requisições em apenas um conjunto do sistema reduzindo possiveis acolamentos, nestas ações o retorno da requisição é feita atraves do **CW-MQTT-GATEWAY**, com a realização de tratamentos para a publicação da resposta processada pelos demais serviçõs do conjunto.

Vale resaltar que o serviçõ criado para coleta de logs tambem interage com os demais serviços que serão apresentados.

<p align="center">
    <img width="70%" src="acao_complexa.png" />
</p>
 - Ação Complexa
 
#### Ação Complexa
Neste conjunto temos um simplificação do sistema em que podemos visualizar os componentes principais das ações consideradas complexas, aquelas em que o processamento da informação e divido em 2 ou mais serviçõs, **CW-CENTRAL-SERVICE** e **CW-MESSAGE-SERVICE**, em que cada um exerce um papel durante o processamento da requisição, sendo **CW-CENTRAL-SERVICE** utilizado para a validação de dados do úsuario e demais informações da requisição, dessa forma garantimos uma menor complexidade no envio das mensagens, sendo **CW-MESSAGE-SERVICE**, apenas utilizado para identificar os membros conectados a um chat em especifico para o recebimento e tratamento de mensagens e para a publicação das mesmas atraves da conexão ao broker.

Optamos por usar MQTT e HTTP no nosso sistema devido às vantagens que cada um desses protocolos oferece em diferentes contextos de comunicação. Cada um tem características que o tornam adequado para determinadas situações, permitindo um equilíbrio ideal entre eficiência e flexibilidade. A seguir vamos explorar as vantagens de cada um desse protocolos e como eles são aplicados em nosso contexto:

### MQTT: Eficiência e Escalabilidade para IOT
O MQTT (Message Queuing Telemetry Transport) é um protocolo de comunicação baseado no modelo publish/subscribe (publicar/assinar), o que o torna altamente eficiente em ambientes com múltiplas conexões simultaneas e idenpendetes. Utilizamos este protocolo em nosso projeto por conta de sua operação ser realizada mesmo em ambientes instaveis ou de baixa largura de banda, atendo de forma eficiente as demandas de conexões entre os microcontroladores e as aplicações da rede.
 - Protocolo leve que utiliza pacotes pequenos para trafegar informações entre a inscricao e a assinatura.
 - Facilidade na integração com plataformas diferentes.

### REST: Simplicidade e Flexibilidade para Integração
O REST (Representational State Transfer) é um estilo de arquitetura amplamente utilizado para a construção de APIs e serviços distribuidos, sendo baseado em operaçãoes simples de requisição HTTP (GET, POST, PUT, DELETE) e no conceito da identificação de recursos por URLs disponibilizadas pelo serviço. Sendo uma vantagem quando a necessidade da simplificação de iterações entre os serviços. 

    - Simplicidade nas chamadas.
    - Ubidquidade na nomeação das URLs de acesso, promovendo uma linguagem familiar no consumo de recursos das APIs.


Optamos por usar REST e MQTT no sistema devido às suas vantagens distintas em termos de simplicidade, flexibilidade e eficiência em diferentes cenários de comunicação. Embora ambos sejam protocolos amplamente utilizados, eles atendem a necessidades específicas e têm características que os tornam ideais para contextos diferentes.
O REST sendo utilizado neste caso para a simplificação da integração aos serviços que não necessitam de esposição ao contexo direto do úsuario, facilitando o desacoplamento e a construção de modulos idenpendetes. Para os casos em que a coneão externa é necessario como no caso de rotornos ao cliente, o protocolo MQTT compre está função, se responsabilizando pela publicação e distribuição dos eventos.


### Exemplo do funcionamento das conexões

Quando trabalhamos com MQTT, cada publicação possui um tópico que define a assinatura das operações que ele realiza para os outros microsserviços inscritos no broker.
Nesta publicação definimos parametros que serão utilizados pelos demais serviços durante o processamento e qual o evento acionado e a identificação do usuario e dispositivo de acesso utilizado.
O exemplo a seguir uma execução de evento de login do nosso microsserviço **CW-BFF-SERVICE** ou de um de nossos microcontroladores conectados. Nele, definimos que esse microsserviço realiza uma requisição `login`. Para chamar essa função devemos passar como parâmetro de entrada um objeto contendo os dados de acesso do usuario (`payload`). Após sua execução, a função retorna como resultado uma outra publicação com o topico (`server`), seguido pela (`session_id`) do usuario e o (`evento`), com payload o token gerado para essa sessão.

<p align="center">
    <img width="70%" src="exemplo_publish_mqttExplorer_edit.png" />
</p>
-- Publicação de login do lado do cliente

<p align="center">
    <img width="70%" src="exemplo_publish_mqttExplorer_server.png" />
</p>
-- Publicacao de login do lado do servidor

Em MQTT, as publicações são formadas por um conjunto de parametros denominados topico (`topic`), que definimos com parametros como a identificação dos dispositivos, sessão da chamada e ação executada, dessa forma conseguimos criar uma estrutura flexivel e compativel com os requisitos dos multiplos sistemas da rede. O tramento destas requisições ocorre em **CW-MQTT-SERVICE**, como no seguinte exemplo:

<p align="center">
    <img width="70%" src="mqtt_py.png" />
</p>

Podemos observer que atraves do metodo (`on_message`), executado sempre que recebe uma nova mensagem, realizamos a chamada para o (`TopicManager`), onde realizamos a deserialização da publicação recebida e envimos e excutamos os processos necessarios para o processamento do evento.

<p align="center">
    <img width="100%" src="topic_manager.png" />
</p>
-- TopicManager

Desse forma centralizamos a idenficação dos eventos e possiveis erros.
As requisições para **CW-CENTRAL-SERVICE** seguem o mesmo processo em todos os eventos, de forma simplificada, a adição de um novo evento depende apenas da alteração em (`TopicManager`) e seu tratamento especifico, embora sua requisição permaça igual aos demais.

<p align="center">
    <img width="100%" src="integracao_central.png" />
</p>
-- integracao_central.png


Apos a requisição, o **CW-CENTRAL-SERVICE** incia o processamento da ação realizada, neste caso se trata da primeira conexão do cliente á aplicação, sendo necessario a geração de um token que sera retornado e utilizado pelo cliente nos demais eventos executados. Esse token tem um prazo definido e seu criação e resposabilidade do **Keycloak**, tornando a idenficação do cliente mais segura dentro do sistema.

<p align="center">
    <img width="100%" src="user_route.png" />
</p>
-- user_route.png

Para uma melhor visualização dos processos que ocorrem durante o tratamento de um evento, utilizamos o padrão **Composer**, utilizado em todas as ações deste serviços e presente em grande parte das aplicações do conjunto.

<p align="center">
    <img width="100%" src="user_login_composer.png" />
</p>
-- user_login_composer.png

A requisição para **CW-AUTH-SERVICE**, executada principalmente nas ações de (`login`) e (`register`), acontece de forma semelhante ao procedimento realizado para a chamadas para **CW-CENTRAL-SERVICE** presente em **CW-MQTT-GATEWAY**:

<p align="center">
    <img width="100%" src="auth_request.png" />
</p>
-- auth_request.png

Esses são os procedimentos basicos utilizados nas requisições simples, em que a resposta ao usuario conta apenas com a utilização principal do **CW-CENTRAL-SERVICE**, sendo entregue o retorno atraves das funções executadas em sequencia entre os serviços.

Nas ocasioes em que o evento utiliza ações complexas, o procedimento deve prosseguir atraves da execução de **CW-MESSAGE-SERVICE**, como no seguinte exemplo em que um cliente conectado realiza a operação de envio de mensagem, os processos descritos anteriormente são mantidos tendo como difereça principal a utilização deste serviço para a publicação da mensagem.

<p align="center">
    <img width="100%" src="forward_message.png" />
</p>
-- forward_message.py

Quando um requisição é recebida por **CW-MESSAGE-SERVICE** ela pode executar as tres seguintes ações:

    - /chat/join - Usuario entra no chat de um grupo em que esta cadastrado
    - /chat/leave - Saida de um usuario do chat de um grupo
    - /chat/send - Envio de mensagem

<p align="center">
    <img width="100%" src="actions_routes.png" />
</p>
-- actions_routes.py

As ações de (`join`) e (`leave`), possuem um funcionamento inverso, uma vez que um usuario envia uma ação de entrada em um chat, o evento é processado atraves do metodo (`join_composer`).

<p align="center">
    <img width="100%" src="join_composer.png" />
</p>
-- join_composer.py

Em que realiza a execuação dos metodos de iniciação do tratamento da ação enque os dados da requisção são submetidos ao bando de dados **CW_MESSAGE_DB**, na tabela **Sessions** afim de cadastrar o chat em que o usuario necessita receber as mensagens presentes na tabela **Messages**, e novas publicações. Quando a ação (`leave`) ocorre, o processo inverso é realizado, deletando os dados da tabela responsavel pela assoaciação da entrada do usuario ao chat.

<p align="center">
    <img width="100%" src="join_leave.png" />
</p>
-- join_leave.png

Para o controle das mensagens enviadas pelos usuarios contamos com uma tabela de armazenamento de mensagens presente em **CW_MESSAGE_DB**, possibilitando o arquivamento destas informações para posterior utilização, o controle destes eventos é realizado principalmente em  (`MessageManager`), em que contamos com métodos utilizados tanto para o envio de novas mensagens como (`forward_message`), para o envio de mensagens ao usuario com a sua entrada no chat (`inbox`) e para a gravação no banco de dados (`save_message`)

<p align="center">
    <img width="100%" src="MessageManager.png" />
</p>
-- MessageManager.png

Dessa forma, quando o usuario entra no chat, temos além do evento de entrada e a possibilidade de enviar mensagens a outros usuarios, a realização do envio das mensagens presentes no grupo anteriormente.

A publicação destas mensagens no broker ocorre atraves do metodo (`message_publish`), tendo como diferenciação a utilização em lote no caso de `inbox` e o envio individual em `forward_message`

<p align="center">
    <img width="100%" src="MessagePublish.png" />
</p>
-- message_publish.png


Agora que explicamos como o processamento das informações ocorre, vamos voltar ao ponto inicial, o que ocorre antes da mensagem chegar ao **Broker**. Um grande ponto de ateção em nosso sistema está em nosso serviço **CW-BFF-SERVICE** e nas possiveis conexões de dispositivos **ESP-8266** encontrados em nossa rede. A forma de construção utilizada para o frontend de nosso sistema e as requisições são feitas utilizando uma conexão **WebSocket** estabelecida entre a página web e o servidor da aplicação, podendo ser uma conexão direta ao servidor local (`CW-BFF-SERVICE`) ou atraves do (`ESP-8266`), ambos os métodos realizam a transformação dos payloads vindos da conexão WebSocket criada no  (`index.html`) 
<p align="center">
    <img width="70%" src="index_socket.png" />
</p>
--  index_socket.png

Esse o `index.html` é enviado para o usuario quando ele realiza requisições para `http://cardinalwave.net`, disponibiniblizado em nossa rede. Neste arquivo utilizamos a tecnica de HTML embutido, onde no mesmo arquivo temos todo o código utilizado para realizar as operações do frontend. Sendo projetado para ser simples e eficiente, permitindo o controle dinâmico da interface. Uma das principais caracteristicas dessa implementação é o uso de um sistema de status para gerenciamento dos modulos a serem exibidos ao usuario. A variavel status e associada as informações do  `localStorage` modulo presente no navegador para o armazenamento de informações pesistentes.
<p align="center">
    <img width="70%" src="index_status.png" />
</p>
-- index_status.png

Quando já logado no sistema, neste caso, já obteve o token gerado por **CW-AUTH-SERVICE**, e armazenado no `localStorage`, ele realiza a chamada por `handleSection`, em que controla as chamadas pelo modulos `chat_list` - Lista de chats/grupos e `chat_join` - a entrada do usuario no chat.
<p align="center">
    <img width="70%" src="index_handleSection.png" />
</p>
-- index_handleSection.png

Ao entrar em um chat o usuario a requisição do úsuario realiza a série de procedimentos e chamadas anteriormente analizadas, tem como resultado a inscricao da seção do úsuario no broker afim de receber as mensagens de um chat de grupo em questão. O recebimento destas mensagens assim como o retorno das chamadas realizadas ocorre de forma semelhante, com a execução dos eventos pelo cliente e a sinalização das seções a serem construidas no frontend, sinalizamos ao script quais devem ser os retornos esperados em cada ocasião, como podemos ver em  `socket.onmessage`, uma função de callback, um método disponibilizado pela instância do WebSocket, uma API nativa do JavaScript.
<p align="center">
    <img width="70%" src="index_socket_message.png" />
</p>
-- index_socket_message.png

Em que agrupamos todas ações que são realizadas pelo o úsuario e necessitam de resposta, com este modo de excecução simplificamos a construção do sistema, embora prejudique a leitura de novos desenvolvedores. A utilização da técnica de HTML embutido em nosso projeto se deve a necessidade de menos sobrecarga na rede e por conta da bibliotecas utilizadas nem nosso microcontroladores que por serem hardwares com foco em baixo consumo, necessitam de menor complexidade para a criação das páginas web.

O tratamento desse payloads quando estamos em uma conexão direta ao servidor, seja na rede principal ou extendida pelo microcontrolador **ESP-32S WROOM**, realiza o envio dos payloads **WebSocket** para o serviço **CW-BFF-SERVICE**, onde ele realiza a criação e gerenciamento da conexão estabelecida e a associa, atraves de id de idenficação unicos, ao id da sessão do usuario.
<p align="center">
    <img width="70%" src="websocket_server.png" />
</p>
--- websocket_server.png

Com essa imagem podemos analizar a utilização de decoradores, disponibilizados atraves da biblioteca **Flaks-Websocket**, onde construimos funções de callback para o tratamento dos eventos vindos da conexão **WebSocket**, em que realizamos o gerenciamento das conexões.

O metodo `handleSection`, acionado quando novas mensagens são enviadas pelo cliente, tem a função de sinalizar o recebimento de um novo pacote para `SessionCompose`, onde centralizamos os envios e recebimentos de ambos os tipos de conexões em que este serviço esta atrelado, **MQTT** e **WebSocket**
<p align="center">
    <img width="70%" src="session_composer.png" />
</p>
--- session_composer.png

Aqui temos um exemplo da utilização do padrão **Strategy**, um dos padrões de projeto comportamentais, aplicado de acordo com as necessidades deste projeto. Strategy permite que você defina uma familia de processos, encapsulados que serão aplicados de acordo com o comportamento necessario para a ocasiao, neste caso sendo chamados por `mqtt`, quando a mensagem foi recebida por meio da inscrição no broker, ou `websocket`, em que a mensagem veio atraves da conexão WebSocket.
<p align="center">
    <img width="70%" src="bff_session_composer.png" />
</p>
--bff_session_composer.png

Quando o ponto de partida da conexão do cliente ocorre no microcontrolador **ESP8266**, o tratamento das requisições e conexões ocorre de maneira diferente por conta das capacidades e objetivos estabelecidos para a utilização deste dispositivo como, por conta de seu baixo poder de processamento comparado a complexidade da aplicação, em que neste caso ele deve alem de manter o maximo possivel de usuarios conectados, ele deve estabelecer anteroirmente uma conexão com a rede do servidor seja, atraves do acesso criado pelo microcontrolador **ESP-32S WROOM**, ou pelo rede central. Essa carga de processamento se torna complexa de lidar nesta situação, por conta disso realizamos parte das operções exercidas em **CW-BFF-SERVICE**, neste dispositivo de forma manual. 
<p align="center">
    <img width="70%" src="esp8266_conn.png" />
</p>
-- esp8266_conn.png

Em nosso microcontrolador **ESP8266**, temos a utilização da estrutura de **main loop**, em que possuimos um método de inicialização (`setup`) e uma função excecutada em de forma constate (`loop`). Em `setup` temos o conjunto de funções utilziadas para a configuração inicial do dispositivo, como a configuração da rede e conexão, e servidores de DNS para processamento das requisições, WebSocket para a conexão da página, Web componente essencial para a operação da aplicação web.
<p align="center">
    <img width="70%" src="esp8266_setup.png" />
</p>
-- esp8266_setup.png

No método `loop` encontramos os metodos utilizados para fazer o controle das requisições, isso é, receber os payloads WebSocket, parsear e identificar as informações necessarias pra compor e realizar a publicação do evento no broker do servidor em que está conectado, sendo grande parte da dificuldade do projeto, por conta da falta de informações disponiveis durante do desenvolvimento do sistema embarcaco, uma caracteristica comum desse tipo de sistema.
<p align="center">
    <img width="70%" src="esp8266_loop.png" />
</p>
-- esp8266_loop.png

As funções de `loop`, são utilzidas além do controle das requisições, mas também para o gerenciamento dos sockets e conexões presentes, atraves de um estrutura desenvolvida pra relacionando o objeto WebSocket com o id da sessão.
<p align="center">
    <img width="70%" src="esp8266_estrutura.png" />
</p>
-- esp8266_estrutura.png

As conexões são gerenciadas atraves das funções de callback realizadas de acordo com o tipo o evento socket `WStype_t`, em `WStype_DISCONNECTED`, `WStype_CONNECTED` e `WStype_TEXT`, sendo `WStype_DISCONNECTED` e `WStype_CONNECTED`, acionados quando recebemos uma nova quesição vinda da página web. 

-- esp8266_socket_controller.png

Quando temos um envio de evento vindo do usuario, atraves de `WStype_TEXT`, em que realizamos uma serie de procedimentos com a inteção de se inscrever no topico da requisição em busca da resposta   

-- esp8266_messageOverMqtt.png

Sendo basicamento o processamento padrão dos eventos recebidos, se analizarmos ocorre de forma semelhante a **CW-BFF-SERVICE**, em que temos um conjunto de funções de callback com a finalidade de reconhecer iterações com o úsuario, como sua solicitação de conexão e desconexão ao servidor WebSocket atraves da página Web, bem como os payloads enviados atraves das ações realizadas na página.


# CardinalCloud: A construção da rede

Nossa rede foi construida utilizando como base conexões Wi-Fi sob o protocolo 802.11g, embora possua a limitação de 54mBps, utilizado dessa forma por conta de ser um protocolo capaz de ser implementado em todos os dispositivos utilizados de forma convencional e pela grande base de documentações e configurações necessarias compativeis. Além da utilização deste protocolo realizamos a criação de servidores web 
