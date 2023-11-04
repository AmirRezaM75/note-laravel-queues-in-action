# Introduction

For a business to be able to handle more traffic, it has three options:
1. Buy more machines (called scaling out).
2. Buy more computing resources for the existing machines (called scaling up).
3. Task software engineers to decrease the resistance coefficient of all the stages of handling a request.

Decreasing the resistance can be achieved in various ways:
1. Code performance optimization.
2. Concurrent processing; running several stages in parallel.
3. Asynchronous execution; removing some stages from the main pipeline and putting it into another pipeline.

## Concurrent Execution

This pattern is called **fan-out, fan-in**. Fanning out is the process of distributing work on multiple processors, and fanning in is combining the results of those processes into one set and passing it to the next stage of the pipeline.
It's possible using Swoole (and its fork Open Swoole) extension.

```php
public function store(Request $request)
{
	$request->user()->can('load_entities');
	[$users, $servers] = Octane::concurrently([
		fn () => User::all(),
		fn () => Supplier::all(),
	]);
	return [$users, $servers];
}
```

## Key components
A queue system has 3 main components:
- queues
- messages
- workers
### Queue
A queue is a linear data structure in which elements can be added to one end, and can only be removed from the other end.
The first message in the queue is the first message that gets processed. (FIFO)
### Messages
A message is a call-to-action trigger.
The message sender doesn't need to worry about what happens after the message is sent, and the message receiver doesn't need to know who the sender is.
### Workers
A worker is a long-running process that dequeues messages and executes the call-to-action.

## Queues in Laravel

```php
Queue::pushRaw('Send invoice #1');
```

Enqueuing raw string message like this is not very useful. Workers won't be able to identify the action that needs to be triggered.

```php
Queue::push(
	new SendInvoice(1)
)
```

Another way of sending messages to the queue that workers can interpret is by using a closure (an anonymous function):

```php
Queue::push(
	fn() => Mail::to($user)->send(new Welcome())
);
```

> Laravel uses the term "push" instead of "enqueue" and "pop" instead of "dequeue"

When you enqueue an object, the queue manager will *serialize* it.
Laravel refers to a message that contains a class instance as a *job*.

Throughout this book, we're going to use the command bus to dispatch our jobs. It allows us to use several functionalities that I'm going to show you later.

```php
dispatch( new SendInvoice(1) );
// Or
Bus::dispatch( new SendInvoice(1) );
// Or
SendInvoice::dispatch(1);
```

To start a worker, you need to run the following artisan command:

```
php artisan queue:work
```

Handling web requests in PHP is done by bootstrapping a fresh application instance exclusively for each request and terminating that instance after the request is handled.
This command will bootstrap **one instance** of your application.
Downsides:
1. A memory leak somewhere in the code will cause the process to consume all available memory in the machine and eventually crash it entirely.
2. Data may leak between jobs if you are not careful.
## Why use a message queue?

### Better User Experience
You don't want your users to wait for certain actions.

### Fault Tolerance
In the case of failure, you can configure your jobs to be retried several times.

### Concurrency
Using Laravel queues, we can achieve concurrency by running multiple workers to process jobs from the queue.

### Scalability
Allocate proportional resources to workers depends on the load.

### Scheduling

```php
$order = Order::create($validated);
sleep(2);
$webhook->send($order);
```

Using queues, you can schedule a specific job to be executed after a specific period.

```php
$order = Order::create($validated);
SendWebhook::dispatch($webhook, $order)->delay(2);
```

### Batching
Batching a large task into several smaller tasks is more efficient.

### Prioritization
If your system can send 1000 emails per hour, you'd want to ensure the most important notifications are sent while less important ones are delayed for later.