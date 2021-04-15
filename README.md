## Guia para Hazelcast com Java

# 1. Introdução
Este é um artigo introdutório sobre Hazelcast, onde veremos como criar um membro do cluster, um mapa distribuído para compartilhar dados entre os nós do cluster e criar um cliente Java para conectar e consultar dados no cluster.

# 2. O que é Hazelcast?
Hazelcast é uma plataforma distribuída In-Memory Data Grid para Java. A arquitetura suporta alta escalabilidade e distribuição de dados em um ambiente em cluster. Ele oferece suporte à descoberta automática de nós e à sincronização inteligente.

Hazelcast está disponível em diferentes edições. Para ver os recursos de todas as edições Hazelcast, podemos consultar o seguinte link: https://docs.hazelcast.com/imdg/latest/#hazelcast-editions. . Neste tutorial, usaremos a edição de código aberto.

Da mesma forma, Hazelcast oferece vários recursos, como Estrutura de Dados Distribuídos, Computação Distribuída, Consulta Distribuída, etc. Para o propósito deste artigo, vamos nos concentrar em um Mapa distribuído.

# 3. Dependência Maven
Hazelcast oferece muitas bibliotecas diferentes para lidar com vários requisitos. Podemos encontrá-los no grupo com.hazelcast em Maven Central.

No entanto, neste artigo, usaremos apenas a dependência principal necessária para criar um membro do cluster Hazelcast independente e o Cliente Hazelcast Java:

```
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast</artifactId>
    <version>4.0.2</version>
</dependency>
```

A versão atual está disponível no repositório central maven.

# 4. Um primeiro aplicativo Hazelcast
### 4.1. Criar um membro Hazelcast
Os membros (também chamados de nós) se unem automaticamente para formar um cluster. Essa junção automática ocorre com vários mecanismos de descoberta que os membros usam para encontrar uns aos outros.

Vamos criar um membro que armazena dados em um mapa distribuído Hazelcast:

```
public class ServerNode {
    
    HazelcastInstance hzInstance = Hazelcast.newHazelcastInstance();
    ...
}
```

Quando iniciamos o aplicativo ServerNode, podemos ver o texto corrido no console, o que significa que criamos um novo nó Hazelcast em nosso JVM que terá que se juntar ao cluster.

```
Members [1] {
    Member [192.168.1.105]:5701 - 899898be-b8aa-49aa-8d28-40917ccba56c this
}
```

Para criar vários nós, podemos iniciar as várias instâncias do aplicativo ServerNode. Como resultado, o Hazelcast criará e adicionará automaticamente um novo membro ao cluster.

Por exemplo, se executarmos o aplicativo ServerNode novamente, veremos o seguinte log no console que diz que há dois membros no cluster.

```
Members [2] {
  Member [192.168.1.105]:5701 - 899898be-b8aa-49aa-8d28-40917ccba56c
  Member [192.168.1.105]:5702 - d6b81800-2c78-4055-8a5f-7f5b65d49f30 this
}
```

### 4.2. Crie um mapa distribuído
A seguir, vamos criar um mapa distribuído. Precisamos da instância de HazelcastInstance criada anteriormente para construir um mapa distribuído que estende a interface java.util.concurrent.ConcurrentMap.

```
Map<Long, String> map = hazelcastInstance.getMap("data");
...
```

Finalmente, vamos adicionar algumas entradas ao mapa:

```
FlakeIdGenerator idGenerator = hazelcastInstance.getFlakeIdGenerator("newid");
for (int i = 0; i < 10; i++) {
    map.put(idGenerator.newId(), "message" + i);
}
```

Como podemos ver acima, adicionamos 10 entradas ao mapa. Usamos FlakeIdGenerator para garantir que obteríamos a chave exclusiva para o mapa. Para obter mais detalhes sobre FlakeIdGenerator, podemos verificar o seguinte link.

Embora este possa não ser um exemplo do mundo real, nós o usamos apenas para demonstrar uma das muitas operações que podemos aplicar ao mapa distribuído. Mais tarde, veremos como recuperar as entradas adicionadas pelo membro do cluster do cliente Hazelcast Java.

Internamente, o Hazelcast particiona as entradas do mapa e distribui e replica as entradas entre os membros do cluster. Para mais detalhes sobre o Hazelcast Map, podemos verificar o seguinte link.

### 4.3. Crie um cliente Hazelcast Java
O cliente Hazelcast nos permite fazer todas as operações Hazelcast sem ser um membro do cluster. Ele se conecta a um dos membros do cluster e delega todas as operações de todo o cluster a ele.

Vamos criar um cliente nativo:

```
ClientConfig config = new ClientConfig();
config.setClusterName("dev");
HazelcastInstance hazelcastInstanceClient = HazelcastClient.newHazelcastClient(config);
```

É simples assim.

### 4.4. Acesse o mapa distribuído do cliente Java
A seguir, usaremos a instância de HazelcastInstance criada anteriormente para acessar o Mapa distribuído:

```
Map<Long, String> map = hazelcastInstanceClient.getMap("data");
...
```

Agora podemos fazer operações em um mapa sem ser um membro do cluster. Por exemplo, vamos tentar iterar as entradas:

```
for (Entry<Long, String> entry : map.entrySet()) {
    ...
}
```

# 5. Configurando Hazelcast
Nesta seção, vamos nos concentrar em como configurar a rede Hazelcast usando declarativamente (XML) e programaticamente (API) e usar o centro de gerenciamento Hazelcast para monitorar e gerenciar nós que estão em execução.

Enquanto o Hazelcast está inicializando, ele procura uma propriedade de sistema hazelcast.config. Se estiver definido, seu valor será usado como caminho. Caso contrário, o Hazelcast procura um arquivo hazelcast.xml no diretório de trabalho ou no caminho de classe.

Se nenhuma das opções acima funcionar, o Hazelcast carrega a configuração padrão, ou seja, hazelcast-default.xml que vem com o hazelcast.jar.

5.1. configuração de rede
Por padrão, Hazelcast usa multicast para descobrir outros membros que podem formar um cluster. Se multicast não for uma forma preferencial de descoberta para nosso ambiente, podemos configurar o Hazelcast para um cluster TCP/IP completo.

Vamos configurar o cluster TCP/IP usando configuração declarativa:

```
<?xml version="1.0" encoding="UTF-8"?>
<hazelcast xmlns="http://www.hazelcast.com/schema/config"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.hazelcast.com/schema/config
                               http://www.hazelcast.com/schema/config/hazelcast-config-4.0.xsd";
    <network>
        <port auto-increment="true" port-count="20">5701</port>
        <join>
            <multicast enabled="false"/>
            <tcp-ip enabled="true">
                <member>machine1</member>
                <member>localhost</member>
            </tcp-ip>
        </join>
    </network>
</hazelcast>
```

Como alternativa, podemos usar a abordagem de configuração Java:

```
Config config = new Config();
NetworkConfig network = config.getNetworkConfig();
network.setPort(5701).setPortCount(20);
network.setPortAutoIncrement(true);
JoinConfig join = network.getJoin();
join.getMulticastConfig().setEnabled(false);
join.getTcpIpConfig()
  .addMember("machine1")
  .addMember("localhost").setEnabled(true);
```

Por padrão, o Hazelcast tentará vincular 100 portas. No exemplo acima, se definirmos o valor da porta como 5701 e limitarmos a contagem de portas a 20, à medida que os membros estiverem ingressando no cluster, o Hazelcast tentará encontrar as portas entre 5701 e 5721.

Se quisermos usar apenas uma porta, podemos desativar o recurso de incremento automático definindo o incremento automático como falso.

### 5.2. Configuração do Centro de Gestão
O centro de gerenciamento nos permite monitorar o estado geral dos clusters, também podemos analisar e navegar nas estruturas de dados em detalhes, atualizar as configurações do mapa e obter o despejo de thread dos nós.

Para usar o Hazelcast Management Center, podemos implantar o aplicativo mancenter-version.war em nosso servidor/contêiner de aplicativos Java ou podemos iniciar o Hazelcast Management Center a partir da linha de comando. Podemos baixar o ZIP Hazelcast mais recente em hazelcast.org. O ZIP contém o arquivo mancenter-version.war.

Podemos configurar nossos nós Hazelcast adicionando a URL do aplicativo da web ao hazelcast.xml e, em seguida, fazer com que os membros Hazelcast se comuniquem com o centro de gerenciamento.

Então, agora vamos configurar o centro de gerenciamento usando a configuração declarativa:

```
<management-center enabled="true">
    http://localhost:8080/mancenter
</management-center>
```

Da mesma forma, aqui está a configuração programática:

```
ManagementCenterConfig manCenterCfg = new ManagementCenterConfig();
manCenterCfg.setEnabled(true).setUrl("http://localhost:8080/mancenter");
```

6. Conclusão
Neste artigo, abordamos conceitos introdutórios sobre Hazelcast. Para obter mais detalhes, podemos dar uma olhada no Manual de Referência: https://docs.hazelcast.org/docs/3.7/manual/html-single/index.html