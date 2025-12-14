Part 2: What Changed with Microservices?

“All of this breaks the moment we move to microservices.”

1 Enter Microservices – The New World

Let’s take your Pizza Delivery System example.

Microservices:

Login Service

Menu Service

Order Service

Payment Service

Delivery Service

Notification Service

Each one is:

A separate application

Deployed independently

Scaled independently

Owned by different teams

2 Logging in Microservices (Chaos Begins)

Each microservice may:

Be written in different languages

Login → Java

Payment → Node.js

Delivery → Python

Use different log formats

Run on different machines / containers

Restart frequently

Example logs:

Login service (Java):

2024-01-10 10:30:01 INFO User authenticated


Payment service (Node.js):

{"time":"10:30:02","level":"error","msg":"Payment failed"}


Delivery service (Python):

ERROR: Delivery partner unavailable

3 Where Are These Logs Stored?

Now ask students:

“Where is the log file?”

Answer:

Not one place

Not one file

Not even one server

In Kubernetes:

Logs live inside containers

Containers are ephemeral

Pods die and come back

Files disappear

So:

/var/log/application.log 


Does NOT exist anymore.

4 The Real Problem (This Is the Key Teaching Moment)

Let’s say:

“Customer says: I placed an order, payment succeeded, but delivery never happened.”

Now answer this question:

 Which service failed?

Possibilities:

Order service?

Payment service?

Delivery service?

Notification service?

In monolith:

One log file → search once

In microservices:

Logs scattered across:

Multiple services

Multiple pods

Multiple nodes

Multiple formats

This is where traditional monitoring fails.

5 Why Simple Log Monitoring Is NOT Enough

Old approach:

Tool → /var/log/application.log


Microservices reality:

Tool → ??? → hundreds of containers → different formats


Challenges:

No single log file

No single format

No fixed server

No single timeline

Hard to correlate requests

6 The Core Question We Need to Answer

At this point, pause and ask the class:

“How do we answer this question in microservices?”

‘What happened to THIS request across ALL services?’

This is the birth of centralized logging.

 Bridge to Next Concept (Perfect Transition)

End this section with:

“Monolith monitoring is about reading a file.
Microservices monitoring is about collecting, standardizing, centralizing, and correlating logs.”

<img width="1024" height="1024" alt="image" src="https://github.com/user-attachments/assets/d5c1b33d-3b60-4e07-883c-6d9bc7947e98" />


