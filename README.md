sudo zookeeper/bin/zkServer.sh start

Open a web browser and type this command http://localhost:8080 . If Storm UI is not starting on 8080 port which is the default port and throwing below error "Exception in thread "main" java.lang.RuntimeException: java.io.IOException: Failed to bind to 0.0.0.0/0.0.0.0:8080" then change the port in “storm/conf/ storm.yaml” configuration file and make it “ui.port: 8081” after that start Strom UI. 


/bin/storm jar examples/examples/storm-example/target/storm-example-1.0-jar-with-dependencies.jar admicloud.storm.eordcount.WordCountTopology WordCount
