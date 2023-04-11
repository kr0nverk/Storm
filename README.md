# Apache Storm

## STEP 1: GET STORM

Давайте посмотрим, как настроить Storm-кластер на вашем локальном компьютере. Загрузите последнюю версию Storm [https://www.apache.org/apache-storm](https://www.apache.org/dyn/closer.lua/storm/apache-storm-2.4.0) и извлеките ее:
```
wget https://www.apache.org/dyn/closer.lua/storm/apache-storm-2.4.0/apache-storm-2.4.0.tar.gz
tar xvf apache-storm-2.4.0.tar.gz
```

## STEP 2: GET ZOOKEEPER

*ПРИМЕЧАНИЕ: В вашей локальной среде должна быть установлена Java 8+.*
Скачайте и запустите ZooKeeper [https://www.apache.org/zookeeper/](https://www.apache.org/dyn/closer.lua/zookeeper/)
```
wget https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.8.1/apache-zookeeper-3.8.1.tar.gz
tar xvf zookeeper-3.8.1.tar.gz
```
Создадим папку Storm и в ней еще две папки Zookeeper и Storm, закинем распакованные файлы в них
```
cp conf/zoo_sample.cfg conf/zoo.cfg
```

## STEP 3: START SERVISES

Apache Storm - это механизм распределенной потоковой обработки. Storm создает ориентированный ациклический граф (DAG), который состоит из вершин графа “spout” и “bolt”, которые управляют потоковой передачей и обработкой данных. Поскольку Storm обрабатывает непрерывные потоковые данные, он настроен на бесконечный запуск до тех пор, пока явно не завершится.

Выполните следующие команды, чтобы запустить все службы в правильном порядке:
```
sudo zookeeper/bin/zkServer.sh start
```

В одном терминале запустите сервер nimbus.
```
./bin/storm nimbus
```
В другом терминале запустите supervisor.

```
./bin/storm supervisor
```
В 3-м терминале запустите веб-интерфейс Storm.
```
./bin/storm ui
```
![Иллюстрация к проекту](https://github.com/kr0nverk/Storm/blob/master/Images/3.png)


Приведенная выше команда запустит пользовательский интерфейс Storm. Вы можете посетить http://localhost:8080 чтобы просмотреть Storm cluster.
Если пользовательский интерфейс Storm не запускается на порту 8080, который является портом по умолчанию, и выдает приведенную ниже ошибку "Exception in thread "main" java.lang.RuntimeException: java.io.IOException: Failed to bind to 0.0.0.0/0.0.0.0:8080" измените порт в “storm/conf/ storm.yaml” конфигурационный файл и напишите “ui.port: 8081” после этого запустите Storm UI. 

![Иллюстрация к проекту](https://github.com/kr0nverk/Storm/blob/master/Images/4.png)

## STEP 3: WORDCOUNT

Word Count - это простой пример потоковой передачи, в котором Storm используется для отслеживания слов и их количества, поступающих в потоковом режиме. Этот пример включен в дистрибутив Storm. Исходный код можно найти в примерах исходного кода.


Теперь откройте другой терминал, чтобы запустить подсчет слов в примере Storm.
Сначала вам нужно его построить

```
git clone https://github.com/ADMIcloud/examples.git
cd examples/storm-example
mvn clean install
```
![Иллюстрация к проекту](https://github.com/kr0nverk/Storm/blob/master/Images/5.png)

Это позволит создать jar-файл внутри целевой папки.

```
cd ../apache-storm-1.0.1
/bin/storm jar examples/examples/storm-example/target/storm-example-1.0-jar-with-dependencies.jar admicloud.storm.eordcount.WordCountTopology WordCount
```
Вы можете просмотреть топологию, зайдя в веб-браузер.

![Иллюстрация к проекту](https://github.com/kr0nverk/Storm/blob/master/Images/6.png)
![Иллюстрация к проекту](https://github.com/kr0nverk/Storm/blob/master/Images/7.png)
![Иллюстрация к проекту](https://github.com/kr0nverk/Storm/blob/master/Images/8.png)
![Иллюстрация к проекту](https://github.com/kr0nverk/Storm/blob/master/Images/9.png)

## STEP 3: DEEP INTO CODE
Генерация предложения и отправка на Splitter
```
public class RandomSentenceSpout extends BaseRichSpout {
  SpoutOutputCollector _collector;
  Random _rand;


  @Override
  public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {
    _collector = collector;
    _rand = new Random();
  }

  @Override
  public void nextTuple() {
    Utils.sleep(100);
    String[] sentences = new String[]{ "the cow jumped over the moon", "an apple a day keeps the doctor away",
        "four score and seven years ago", "snow white and the seven dwarfs", "i am at two with nature" };
    String sentence = sentences[_rand.nextInt(sentences.length)];
    _collector.emit(new Values(sentence));
  }

  @Override
  public void ack(Object id) {
  }

  @Override
  public void fail(Object id) {
  }

  @Override
  public void declareOutputFields(OutputFieldsDeclarer declarer) {
    declarer.declare(new Fields("word"));
  }

}
```
Подсчет слов
```
public static class WordCount extends BaseBasicBolt {
    Map<String, Integer> counts = new HashMap<String, Integer>();

    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {
      String word = tuple.getString(0);
      Integer count = counts.get(word);
      if (count == null)
        count = 0;
      count++;
      counts.put(word, count);
      collector.emit(new Values(word, count));
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
      declarer.declare(new Fields("word", "count"));
    }
}
```
Построенная топология
```
public static void main(String[] args) throws Exception {
    TopologyBuilder builder = new TopologyBuilder();
    builder.setSpout("spout", new RandomSentenceSpout(), 5);
    builder.setBolt("split", new SplitSentence(), 8).shuffleGrouping("spout");
    builder.setBolt("count", new WordCount(), 12).fieldsGrouping("split", new Fields("word"));

    Config conf = new Config();
    conf.setDebug(true);

    if (args != null && args.length > 0) {
      conf.setNumWorkers(3);

      StormSubmitter.submitTopologyWithProgressBar(args[0], conf, builder.createTopology());
    } else {
      conf.setMaxTaskParallelism(3);
      LocalCluster cluster = new LocalCluster();
      cluster.submitTopology("word-count", conf, builder.createTopology());
      Thread.sleep(10000);
      cluster.shutdown();
    }
}
```


Чтобы отключить топологию, используйте следующую команду:
```
./bin/storm kill WordCount
```

## STEP 8: TERMINATE THE STORM ENVIRONMENT
Теперь, когда вы дошли до конца, не бойтесь удалять среду Storm — разрушьте все побыстрее.

- Остановите клиенты сервер nimbus и supervisor с помощью Ctrl-C, если вы еще этого не сделали.
- Остановите UI с помощью Ctrl-C.
- Наконец, остановите сервер ZooKeeper с помощью Ctrl-C.
Чтобы отключить топологию, используйте следующую команду:
```
./bin/storm kill WordCount
```
## CONGRATULATIONS!
Вы успешно запустили Apache Storm. Чтобы узнать больше, можно воспользоваться:

- Прочтите краткое [Введение](https://storm.apache.org/index.html), чтобы узнать, как Storm работает на высоком уровне, ее основные концепции и как она сравнивается с другими технологиями. Чтобы разобраться в Storm более подробно, ознакомьтесь с [Документация](https://storm.apache.org/releases/2.4.0/index.html).
- Просмотрите [Примеры](https://storm.apache.org/about/integrates.html) использования, чтобы узнать, как другие пользователи нашего мирового сообщества извлекают пользу из Storm.
