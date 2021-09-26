---
title: 'Rabbit MQ - Intro'
date: '2021-09-26'
---

Rabbit MQ is pub-sub/message broker. It can be use to send message asyncronously.

Jargon:
- Producer: program that sending message
- Queue: place of message being stored
- Consumer: program wait to receive a message

![Diagram](https://www.rabbitmq.com/img/tutorials/python-one.png)
- P = Producer
- C = Consumer
- Box in the middle = queue

Let's build hello world program
1. Install Go RabbitMQ client library
    ```sh
    go get github.com/streadway/amqp
    ```
1. Run rabbitmq using docker(I prefer)
    ```sh
    docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
    ```
1. Create producer
    <details>
        <summary>send.go</summary>

    ```go
    package main

    import (
    	"log"
    	"os"
    
    	"github.com/streadway/amqp"
    )
    
    func failOnError(err error, msg string) {
    	if err != nil {
    		log.Fatalf("%s: %s", msg, err)
    	}
    }
    
    func main() {
    	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    	failOnError(err, "Failed to connect to RabbitMQ")
    	defer conn.Close()
    
    	ch, err := conn.Channel()
    	failOnError(err, "Failed to open a channel")
    	defer ch.Close()
    
    	q, err := ch.QueueDeclare(
            "hello", // name
            false,   // durable
            false,   // delete when unused
            false,   // exclusive
            false,   // no-wait
            nil,     // arguments
    	)
    	failOnError(err, "Failed to declare a queue")
    
    	arg := os.Args[1]
    	body := string(arg)
    	err = ch.Publish(
            "",     // exchange
            q.Name, // routing key
            false,  // mandatory
            false,  // immediate
    		amqp.Publishing{
    			ContentType: "text/plain",
    			Body:        []byte(body),
    		},
    	)
    	failOnError(err, "Failed to publish a message")
    }
    ```
    </details>
1. Create consumer
    <details>
        <summary>receive.go</summary>

    ```go
    package main

    import (
    	"log"
    
    	"github.com/streadway/amqp"
    )
    
    func failOnError(err error, msg string) {
    	if err != nil {
    		log.Fatalf("%s: %s", msg, err)
    	}
    }
    
    func main() {
    	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    	failOnError(err, "Failed to connect to RabbitMQ")
    	defer conn.Close()
    
    	ch, err := conn.Channel()
    	failOnError(err, "Failed to open a channel")
    	defer ch.Close()
    
    	q, err := ch.QueueDeclare(
            "hello", // name
            false,   // durable
            false,   // delete when unused
            false,   // exclusive
            false,   // no-wait
            nil,     // arguments
    	)
    	failOnError(err, "Failed to declare a queue")
    
    	msgs, err := ch.Consume(
            q.Name, // queue
            "",     // consumer
            true,   // auto-ack
            false,  // exclusive
            false,  // no-local
            false,  // no-wait
            nil,    // args
    	)
    
    	forever := make(chan bool)
    
    	go func() {
    		for d := range msgs {
    			log.Printf("Received a message: %s", d.Body)
    		}
    	}()
    
    	log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
    	<-forever
    }
    ```
    </details>

    Note: We re-declare the queue because if we run receive.go first and send later the queue is already exist.
1. Execute:
    1. go run send "Hello world!"
    1. go run receive.go
1. Stdout should like this on receive.go
    ```
    ➜  RabbitMQ git:(master) ✗ go run receive.go
    2021/09/26 22:30:42  [*] Waiting for messages. To exit press CTRL+C
    2021/09/26 22:30:42 Received a message: Hello World!
    2021/09/26 22:31:39 Received a message: Hello World!
    2021/09/26 22:33:59 Received a message: halo
    2021/09/26 22:34:10 Received a message: nama saya wawan
    ```
1. You can check the queue by execute this
    ```
    docker exec -it rabbitmq rabbitmqctl list_queues    
    ```

You can check my full code [here](https://github.com/ardinusawan/my-learning/tree/master/RabbitMQ)

It based on [official rabbit mq tutorial](https://www.rabbitmq.com/tutorials/tutorial-one-go.html)
