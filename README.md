# Mapear IAM — Neo4J -Graph

![](https://cdn-images-1.medium.com/max/2304/1*uhEr0YiK6Bv95uM9jOy6OQ.jpeg)

Olá pessoal o desafio desta vez consiste em exibir graficamente o que esta contido nas configurações o  **IAM**  da  **AWS**  de uma maneira amigável, vou precisar de uma ferramenta para me ajudar a entender e debater com outras áreas, criar os modelos na mão perderíamos a velocidade em obter os melhores questionamentos, que eu entendo em melhores perguntas e respostas e assim obteremos os melhores resultados, então vamos lá:

A ideia principal é extrair os modelos via  **aws-cli**  e realizar a ingestão para a ferramenta.

Utilizo Linux e este tutorial foi realizado utilizando o Ubuntu 18.04.

Abaixo vou mencionar brevemente sobre cada tecnologia utilizada a fim de esclarecer para alguns ou reforçar o conhecimento para quem tem já sabe.

### IAM

![](https://cdn-images-1.medium.com/max/720/1*RH4MIaJWh_AxKA77PYgPFg.png)

Segue a definição da própria AWS:

> O AWS Identity and Access Management (IAM) permite que você gerencie com segurança o acesso aos serviços e recursos da AWS. Usando o IAM, você pode criar e gerenciar usuários e grupos da AWS e usar permissões para conceder e negar acesso a recursos da AWS.

Trocando em miúdos é a camada de gerenciamento do usuário AWS, onde criamos o acesso e concedemos o acesso, e criamos as regras de acesso para que cada um dos usuários possuam o acesso de acordo com a sua necessidade.

**Segue uma máxima e sempre atual do meu xará William  _Deming:_**

> Não se gerencia o que não se mede, não se mede o que não se define, não se define o que não se entende e não há sucesso no que não se gerencia.

### AWS CLI

![](https://cdn-images-1.medium.com/max/720/1*AGtFSyzqRespUiBa5oZ04Q.jpeg)

Utilizaremos o aws-cli para obter as informações provenientes do AWS IAM, coletaremos de forma programática e realizaremos a carga para dentro do  **Neo4J**. Para configurar o cliente segue as instruções neste link:  [https://docs.aws.amazon.com/pt_br/cli/latest/userguide/cli-chap-configure.html](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/cli-chap-configure.html)

Feito a configuração vamos realizar o export:

```
$ aws iam get-account-authorization-details > iam_account.json
```

Pronto se não apareceu nenhuma mensagem de erro ocorreu tudo com sucesso. A saída é um arquivo  **json**  que permite ser inspecionado, faça isso. Aqui eu costumo fazer assim:

```
$ cat iam_account.json | jq
```

O  **jq**  ajuda a validar o  **json**, vai mostrar o resultado com sintaxe destacada. Caso não possua em seu sistema operacional instale, aqui eu faço assim:

```
$ sudo apt install -y jq
```

Se conseguiu visualizar o conteúdo, esta tudo certo, vamos seguir.

### Docker

![](https://cdn-images-1.medium.com/max/720/1*uoHYxwC8IPQhjmVK0LXM2w.png)

Fonte: docker.com

Necessário que tenha o docker instalado, segue passo a passo:  [https://www.digitalocean.com/community/tutorials/como-instalar-e-usar-o-docker-no-ubuntu-18-04-pt](https://www.digitalocean.com/community/tutorials/como-instalar-e-usar-o-docker-no-ubuntu-18-04-pt)

Também será necessário a instalação do docker-compose, seguir este link:  [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

Poderá ser utilizado a imagem oficial do dockerhub:  [https://hub.docker.com/_/neo4j](https://hub.docker.com/_/neo4j), hoje a versão é a 3.5.3.

Mas nós iremos iremos construir uma imagem, e já incluir o plugin APOC já habilitado, eu mesmo demorei um pouco até entender isso, as informações existem mas ainda não estão todas com o passo a passo bem mapeados, então quando a documentação se torna insuficiente cria sua própria.

Antes disso vamos trazer a dependências, executar o script:  **01-dependencias.sh**, que possui a finalidade de criar um dietório e realizar o download neste diretório denominado:  **plugins.**

```
$ bash 01-dependencias.sh
```

Ao concluir o script, será listado o arquivo:  **apoc-3.5.0.2-all.jar**

Para construirmos a nossa própria imagem utilizaremos o arquivo:  **Dockerfile**, então vamos executar o script:  **02-construir-imagem-docker.sh**e informar como parâmetro o nome da imagem, nesta POC foi definido o nome:  **nome-sua-imagem**.

```
$ bash 02-construir-imagem-docker.sh nome-sua-imagem
```

Não deve apresentar nenhuma mensagem de erro, aqui passou liso. O resultado obtido será:

**REPOSITORY                         TAG                 IMAGE ID            CREATED                  SIZE**  
nome-sua-imagem                    latest              5cd78ec7247d        Less than a second ago   217MB

Agora vamos inicializar através do docker-compose, desta maneira:

```
$ docker-compose up -d
```

A resposta é isso:

Creating network "neo4j_analytics" with the default driver  
Creating neo4j-apoc ...   
Creating neo4j-apoc ... done

E através do comando:  **docker ps**  poderá acompanhar a execução:

    $ docker ps  
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                      NAMES  
    01d6df12ef38        nome-sua-imagem     "/sbin/tini -g -- /d…"   46 seconds ago      Up 41 seconds       0.0.0.0:7474->7474/tcp, 7473/tcp, 0.0.0.0:7687->7687/tcp   neo4j-apoc

Contêiner em pé, vamos copiar os dados obtidos através do  **aws-cli,** lá para dentro do contêiner.

Vamos copiar o arquivo:  `**iam_account.json**,`

    $ docker cp iam_account.json neo4j-apoc:/var/lib/neo4j/import/iam_account.json

Concluído essa parte do docker

### Neo4J

![](https://cdn-images-1.medium.com/max/720/1*vbs-yP6PB4id55syTwETTg.png)

fonte: neo4j.com

Aqui é onde acontece a mágica, segue também a definição do próprio fabricante:

> Uma plataforma gráfica que revela e persiste as conexões.

> A plataforma gráfica utiliza uma abordagem de conexões aos dados. Amplia a capacidade de reconhecer a importância de relacionamentos e conexões persistentes em todas as transições da existência: a ideia é projetar em um modelo lógico, e implementar em um modelo físico, usando uma linguagem de consulta e persistência dentro de modo escalável e confiável em um sistema de banco de dados. A base da representação dos dados conectados é conhecida como  [**gr**](https://neo4j.com/why-graph-databases/?ref=product)**aph.**

Agora vamos colocar a mão na ferramenta, até então foi somente a preparação e entendimento do ambiente.

Vá até o browser e digite  [http://localhost:7474/browser/](http://localhost:7474/browser/), você vai se deparar com o primeiro acesso, usuário e senha padrão, são:  **neo4j**,  **neo4j** respectivamente. após esse passo você irá informar o usuário e senha definitivo.

![](https://cdn-images-1.medium.com/max/1080/1*QOIQEPxFLtoR8sSdBlv8fg.png)

Vamos verificar a versão do plugin APOC, segue snapshot abaixo:

```
RETURN apoc.version();
```

No canto superior a sua direita há uma seta, ao posicionar o curso do mouse a descrição é play, clique lá.

![](https://cdn-images-1.medium.com/max/1080/1*67hLzH5TEcD3grTajVC5sA.png)

A versão obtida foi: “**3.5.0.2**”

A ferramenta com a interface gráfica, funciona dessa maneira, digitar no prompt o comando e clicar em play.

![](https://cdn-images-1.medium.com/max/1080/1*4sNIKE99TUr58IZLRqd8kQ.png)

O próximo passo é realizar a ingestão dos dados obtidos através do  **aws-cli**. o caminho completo do arquivo dentro do contêiner ficou definido como:  **/var/lib/neo4j/import/iam_account.json**

Abaixo segue as queries para realizarmo a ingestão manual através da interface gráfica.

    // POLICIES  
    CALL apoc.load.json("/var/lib/neo4j/import/iam_account.json",  
    '.Policies[*]') YIELD value as row  
    MERGE(p:IAM_Policy {id:row.PolicyId, name:row.PolicyName, arn:row.Arn})

    // GROUPS  
    CALL apoc.load.json("/var/lib/neo4j/import/iam_account.json",  
    '.GroupDetailList[*]') YIELD value as row  
    MERGE(p:IAM_Group {id:row.GroupId, name:row.GroupName, arn:row.Arn})  
  
    // Load User and relationship to Groups,Policies  
    CALL apoc.load.json("/var/lib/neo4j/import/iam_account.json",  
    '.UserDetailList[*]') YIELD value as row  
    UNWIND row.AttachedManagedPolicies as policy  
    UNWIND row.GroupList as group  
    CREATE (u:IAM_User {id:row.UserId,name:row.UserName,arn:row.Arn})  
    WITH u,policy, group  
    MATCH (p:IAM_Policy {name:policy.PolicyName})  
    CREATE (u)-[:HAS_POLICY]->(p)  
    WITH u,p, group  
    MATCH (g:IAM_Group {name:group})  
    CREATE (u)-[:MEMBER_OF]->(g)  

    // Load Roles and attached Policies to the Roles  
    CALL apoc.load.json("/var/lib/neo4j/import/iam_account.json",  
    '.RoleDetailList[*]') YIELD value as row  
    UNWIND row.AttachedManagedPolicies as policy  
    UNWIND policy as pol  
    WITH row.RoleName as role, row.Path as path, row.Arn as arn, pol  
    MERGE(r:IAM_Role {name:role, path:path, arn:arn})  
    MERGE(p:IAM_Policy {name:pol.PolicyName,arn:pol.PolicyArn})  
    MERGE(r)-[:HAS_POLICY]->(p)  

    // Create relationship btw Groups and Policies  
    CALL apoc.load.json("/var/lib/neo4j/import/iam_account.json",  
    '.GroupDetailList[*]') YIELD value as row  
    UNWIND row.AttachedManagedPolicies as policy  
    MATCH (g:IAM_Group {name:row.GroupName})  
    MATCH (p:IAM_Policy {name:policy.PolicyName})  
    CREATE (g)-[:HAS_POLICY]->(p)  

    // Create Policy Actions and relate it to the Policy  
    CALL apoc.load.json("/var/lib/neo4j/import/iam_account.json") YIELD value as row  
    UNWIND row.Policies as p  
    UNWIND p.PolicyVersionList as a  
    UNWIND a.Document as d  
    UNWIND d.Statement as s  
    UNWIND s.Action as act  
    WITH p.PolicyName as pol, act as action  
    MATCH (p1:IAM_Policy {name:pol})  
    MERGE(a1:IAM_Policy_Action {action:action})  
    MERGE(p1)-[:HAS_ACTION]->(a1) 
 
    // Create Policy Resources and relate it to the Policy  
    CALL apoc.load.json("/var/lib/neo4j/import/iam_account.json") YIELD value as row  
    UNWIND row.Policies as p  
    UNWIND p.PolicyVersionList as a  
    UNWIND a.Document as d  
    UNWIND d.Statement as s  
    UNWIND s.Resource as res  
    WITH p.PolicyName as pol, res as resource  
    MATCH (p1:IAM_Policy {name:pol})  
    MERGE(r1:IAM_Policy_Resource {name:resource})  
    MERGE(p1)-[:HAS_RESOURCE]->(r1)  

    // Load Services   
    CALL apoc.load.json("/var/lib/neo4j/import/iam_account.json",  
    '.RoleDetailList[*]') YIELD value as row  
    UNWIND row.AssumeRolePolicyDocument as arpd  
    UNWIND arpd.Statement as stmt  
    WITH stmt,row.RoleName as role  
    WHERE stmt.Effect = 'Allow'   
    UNWIND keys(stmt.Principal) AS key  
    WITH role,key, stmt.Principal as princ  
    WHERE key = 'Service'  
    MATCH(r:IAM_Role {name:role})  
    CREATE(s:AWS_Service {name: princ[key] })  
    CREATE (s)-[:CAN_ASSUME_ROLE]->(r)  

 
Efetuado a ingestão, para obter o acesso ir até o primeiro icone superior a sua esquerda, com o nome:  **Database Information**, abaxo sera diposta a lista de tags, conforme os carregamentos, contendo as seguintes informações:

-   **AWS_Service**
-   **IAM_Group**
-   **IAM_Policy**
-   **IAM_Policy_Action**
-   **IAM_Policy_Resource**
-   **IAM_Role**
-   **IAM_User**

Escolher uma das views, e selecionar o modo **graph**, você também poderá utilizar o modo **fullscreen**.

![](https://cdn-images-1.medium.com/max/1080/1*YXm5uD4xvm0ZBQfSxs_zPA.png)

Parabéns a você que chegou até aqui.

### Referências

-   [https://neo4j.com/](https://neo4j.com/)
-   [http://bit.ly/dl-neo4j](http://bit.ly/dl-neo4j)
-   [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)
-   [https://github.com/rnrbarbosa/aws-iam-neo4j](https://github.com/rnrbarbosa/aws-iam-neo4j)
-   [https://github.com/neo4j-contrib/neo4j-apoc-procedures](https://github.com/neo4j-contrib/neo4j-apoc-procedures)
-   [https://www.digitalocean.com/community/tutorials/como-instalar-e-usar-o-docker-no-ubuntu-18-04-pt](https://www.digitalocean.com/community/tutorials/como-instalar-e-usar-o-docker-no-ubuntu-18-04-pt)

### FIM
