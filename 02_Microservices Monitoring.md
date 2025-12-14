

# Part 2: What Changed with Microservices?

> **“Everything we discussed so far works beautifully — until we move to microservices.”**

This is the moment where **traditional monitoring breaks down**.

---

## 2.1 Enter Microservices — The New World

Let’s take a **Pizza Delivery System** as an example.

Instead of one big application, the system is broken into **multiple small services**:

* Login Service
* Menu Service
* Order Service
* Payment Service
* Delivery Service
* Notification Service

Each service is designed to do **one specific job**.

Now, very importantly, **each microservice is**:

* A **separate application**
* **Deployed independently**
* **Scaled independently**
* Often **owned by a different team**

So instead of *one application*, we now have **many small applications working together**.

---

## 2.2 Logging in Microservices — Where Chaos Begins

In a microservices world, logging is no longer uniform.

Each microservice may:

* Be written in **different programming languages**

  * Login → Java
  * Payment → Node.js
  * Delivery → Python

* Use **different logging libraries**

* Generate logs in **different formats**

* Run in **different containers**

* Restart frequently

### Example log outputs:

**Login Service (Java):**

```
2024-01-10 10:30:01 INFO User authenticated
```

**Payment Service (Node.js):**

```json
{"time":"10:30:02","level":"error","msg":"Payment failed"}
```

**Delivery Service (Python):**

```
ERROR: Delivery partner unavailable
```

Already, we can see:

* Different formats
* Different structures
* Different levels of detail

---

## 2.3 Where Are These Logs Stored?

Now pause and ask the students:

> **“Where is the log file?”**

In monolithic applications, the answer was easy:

```
/var/log/application.log
```

But in microservices, the reality is very different.

### The truth is:

* Logs are **not in one place**
* Logs are **not in one file**
* Logs are **not on one server**

In Kubernetes:

* Logs live **inside containers**
* Containers are **ephemeral**
* Pods die and restart
* Log files disappear when pods disappear

So the familiar idea of:

```
/var/log/application.log
```

**does not exist anymore.**

---

## 2.4 The Real Problem 

Now present this scenario to the class:

> **“A customer says: I placed an order, payment succeeded, but delivery never happened.”**

Ask the students:

**Which service failed?**

Possible answers:

* Order Service?
* Payment Service?
* Delivery Service?
* Notification Service?

### In a monolith:

* One application
* One log file
* One place to search

### In microservices:

* Logs are scattered across:

  * Multiple services
  * Multiple pods
  * Multiple nodes
  * Multiple formats

This is the moment where **traditional monitoring completely fails**.

---

## 2.5 Why Simple Log Monitoring Is No Longer Enough

### Old monitoring approach:

```
Monitoring Tool → /var/log/application.log
```

### Microservices reality:

```
Monitoring Tool → ??? → hundreds of containers → different formats
```

### Core challenges:

* No single log file
* No single log format
* No fixed server
* No unified timeline
* Very hard to trace a single request end-to-end

---

## 2.6 The Core Question We Must Answer

Pause here.
This is the most important question in the entire topic.

Ask the class:

> **“In a microservices system, how do we answer this?”**

> **“What happened to THIS request across ALL services?”**

This question **cannot** be answered by traditional logging.

This question is what **gave birth to centralized logging and observability platforms**.

---

## Bridge to the Next Concept (Perfect Transition)

End this section with this statement:

> **“Monolith monitoring is about reading a file.”**
> **“Microservices monitoring is about collecting, standardizing, centralizing, and correlating logs.”**

This naturally leads to:

**Why we need EFK / ELK stacks**
**Why log aggregation is mandatory**
**Why Fluent Bit, Elasticsearch, and Kibana exist**

---




<img width="2848" height="1600" alt="image" src="https://github.com/user-attachments/assets/bb6259ce-721f-4470-8714-4a4172c83c8f" />



