# Canceling Abandoned Orders

Now let's send an SMS message every 15 minutes until the user completes the checkout, or we cancel the order after 1 hour.

```php
MonitorPendingOrder::dispatch($order)
	->delay(900);
```

We can also set the delay using a `DateTimeInterface` implementation:

```php
MonitorPendingOrder::dispatch($order)->delay(
	now()->addMinutes(15)
);
```

Every time the worker picks up the job, it'll increment the attempts counter. And given that jobs are allowed to be attempted only a single time by default, we need to make sure our job has enough `$tries` to run four times.

```php
class MonitorPendingOrder implements ShouldQueue
{
	public $tries = 4;
	
	public function handle()
	{	
		if ($this->order->created_at->lt(now()->subHour())) {
			$this->order->cancel();
			return;
		}
	
		SMS::send(...);
	
		// put the job back in the queue to be retried later.
		$this->release(
			now()->addMinutes(15)
		);
	}
}
```

> Warning: Using the SQS driver, you can only delay a job for 15 minutes.

As we've mentioned, there's no guarantee that workers will pick the job up after the delay or backoff period passes. If the queue is busy and not enough workers are running, the `MonitorPendingOrder` job in this example may not run enough times to send the three SMS reminders before canceling the order.

```
00:00 => Dispatched
00:15 => 1st Run
00:50 => 2nd Run
00:05 => 3rd Run (The order gets cancelled.)
```