## Introdução ao Hazelcast Jet


# 1. Introdução
Neste tutorial, aprenderemos sobre Hazelcast Jet. É um mecanismo de processamento de dados distribuído fornecido pela Hazelcast, Inc. e é construído com base no Hazelcast IMDG.

# 2. O que é Hazelcast Jet?
Hazelcast Jet é um mecanismo de processamento de dados distribuído que trata os dados como fluxos. Ele pode processar dados armazenados em um banco de dados ou arquivos, bem como os dados que são transmitidos por um servidor Kafka.

Além disso, ele pode executar funções de agregação em fluxos de dados infinitos, dividindo os fluxos em subconjuntos e aplicando agregação em cada subconjunto. Esse conceito é conhecido como janelamento na terminologia do Jet.

Podemos implantar o Jet em um cluster de máquinas e, em seguida, enviar nossos trabalhos de processamento de dados a ele. O Jet fará com que todos os membros do cluster processem os dados automaticamente. Cada membro do cluster consome uma parte dos dados e isso facilita o dimensionamento para qualquer nível de taxa de transferência.

Aqui estão os casos de uso típicos para Hazelcast Jet:

- Processamento de fluxo em tempo real;
- Processamento de lote rápido;
- Processamento de Java 8 Streams de forma distribuída;
- Processamento de dados em microsserviços.

# 3. Configuração
Para configurar o Hazelcast Jet em nosso ambiente, precisamos apenas adicionar uma única dependência Maven ao nosso pom.xml.

Veja como fazemos:

```
<dependency>
    <groupId>com.hazelcast.jet</groupId>
    <artifactId>hazelcast-jet</artifactId>
    <version>4.2</version>
</dependency>
```

Incluir essa dependência fará o download de um arquivo jar de 10 Mb que nos fornece toda a infraestrutura de que precisamos para construir um pipeline de processamento de dados distribuído.

A versão mais recente do Hazelcast Jet pode ser encontrada aqui.

# 4. Aplicativo de amostra
Para aprender mais sobre o Hazelcast Jet, criaremos um aplicativo de exemplo que recebe uma entrada de frases e uma palavra para localizar nessas frases e retorna a contagem da palavra especificada nessas frases.

### 4.1. O Pipeline
Um Pipeline forma a construção básica de um aplicativo Jet. O processamento em um pipeline segue estas etapas:

- ler dados de uma fonte;
- transformar os dados;
- escrever dados em uma pia.

Para nosso aplicativo, o pipeline fará a leitura de uma lista distribuída, aplicará a transformação de agrupamento e agregação e, finalmente, gravará em um mapa distribuído.

Veja como escrevemos nosso pipeline:

```
private Pipeline createPipeLine() {
    Pipeline p = Pipeline.create();
    p.readFrom(Sources.<String>list(LIST_NAME))
      .flatMap(word -> traverseArray(word.toLowerCase().split("\\W+")))
      .filter(word -> !word.isEmpty())
      .groupingKey(wholeItem())
      .aggregate(counting())
      .writeTo(Sinks.map(MAP_NAME));
    return p;
}
```

Depois de ler a fonte, percorremos os dados e os dividimos no espaço usando uma expressão regular. Depois disso, filtramos os espaços em branco.

Por fim, agrupamos as palavras, agregamos e escrevemos os resultados em um Mapa.

### 4.2. O emprego
Agora que nosso pipeline está definido, criamos um trabalho para executar o pipeline.

Veja como escrevemos uma função countWord que aceita parâmetros e retorna a contagem:

```
public Long countWord(List<String> sentences, String word) {
    long count = 0;
    JetInstance jet = Jet.newJetInstance();
    try {
        List<String> textList = jet.getList(LIST_NAME);
        textList.addAll(sentences);
        Pipeline p = createPipeLine();
        jet.newJob(p).join();
        Map<String, Long> counts = jet.getMap(MAP_NAME);
        count = counts.get(word);
        } finally {
            Jet.shutdownAll();
      }
    return count;
}
```

Criamos uma instância Jet primeiro para criar nosso trabalho e usar o pipeline. Em seguida, copiamos a lista de entrada para uma lista distribuída para que esteja disponível em todas as instâncias.

Em seguida, enviamos um trabalho usando o pipeline que construímos acima. O método newJob () retorna um trabalho executável que é iniciado pelo Jet de forma assíncrona. O método de junção aguarda a conclusão do trabalho e lança uma exceção se o trabalho for concluído com um erro.

Quando o trabalho é concluído, os resultados são recuperados em um mapa distribuído, conforme definimos em nosso pipeline. Portanto, obtemos o Mapa da instância Jet e obtemos as contagens da palavra em relação a ele.

Por último, encerramos a instância Jet. É importante desligá-lo após o término de nossa execução, pois a instância do Jet inicia seus próprios threads. Caso contrário, nosso processo Java ainda estará ativo mesmo depois que nosso método for encerrado.

Aqui está um teste de unidade que testa o código que escrevemos para o Jet:

```
@Test
public void whenGivenSentencesAndWord_ThenReturnCountOfWord() {
    List<String> sentences = new ArrayList<>();
    sentences.add("The first second was alright, but the second second was tough.");
    WordCounter wordCounter = new WordCounter();
    long countSecond = wordCounter.countWord(sentences, "second");
    assertEquals(3, countSecond);
}
```

# 5. Conclusão
Neste artigo, aprendemos sobre Hazelcast Jet. Para saber mais sobre ele e seus recursos, consulte o manual: https://jet-start.sh/docs/get-started/intro