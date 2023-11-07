# Canceling a Conference

```php
class CancelConference implements ShouldQueue
{
	public function __construct(
		private Collection $attendees
	) {}

	public function handle()
	{
		$this->attendees
			->each(function($attendee) {
				$attendee->invoice->refund();
				Mail::to($attendee)->send(...);
			});
	}
}
```

## Handling Timeouts

Since this is a huge conference, the `CancelConference` job may need several minutes to complete. However, Laravel has a timeout mechanism to prevent jobs from running indefinitely, as this may cause workers to get stuck and not process any job. This limit is set to **60 seconds** by default.

```php
class CancelConference implements ShouldQueue
{
	public $timeout = 18000;
}
```

Or

```
php artisan queue:work --timeout=18000
```

Under the hood, Laravel uses a PHP process control extension called **[PCNTL](https://www.php.net/manual/en/function.pcntl-alarm.php)** to install a timer inside the worker process. This timer starts counting once a job is picked up by the worker and sends a **[SIGALRM](https://www.php.net/manual/en/function.pcntl-signal.php)** signal to the worker process. Laravel handles this signal and **kills the worker** process along with the job it's processing.

## Preventing Job Duplication

When a worker picks a job up from the queue, it marks it as **reserved** so no other worker picks that same job up. Then, once it finishes processing the job, it either removes it from the queue or releases it back to be retried. With that in mind, there's a risk that if the worker process crashes while in the middle of processing a job, this job will remain reserved forever and will never run.

To avoid this, Laravel sets a timeout for how long a job may remain in a reserved state. This timeout is **90 seconds** by default, and it's set inside the `config/queue.php` configuration file on the connection level:

```php
return [
	'connections' => [
		'database' => [
			// ...
			'retry_after' => 90,
		],
	];
];
```

Now our job may run for 5 hours, and Laravel will remove the reserved state in 90 seconds. That means another worker can pick this same job up while it's still being processed. This will lead to job duplication.

> To avoid job duplication for long-running jobs, we have to always make sure the value of `retry_after` is more than any timeout we set on a worker level or a job level

## Using a Separate Connection

We had to set `retry_after` to 18060 a bit earlier. This will apply to all jobs dispatched to our database queue connection. So if a worker crashes while it's in the middle of processing any job, other workers will not pick it up for 5 hours.

```php
return [
	'connections' => [
		'database-cancelations' => [
			// ...
			'retry_after' => 18060,
		],
	];
];
```

Here's how to dispatch jobs to the new connection:

```php
CancelConference::dispatch($conference)
	->onConnection('database-cancelations');
```

Or

```php
class CancelConference implements ShouldQueue
{
	public $connection = 'database-cancelations';
}
```

Here's how to consume jobs from our new connection:

```
php artisan queue:work database-cancelations
```

> Warning: Workers cannot switch between connections.

## Fault Tolerance Using Checkpointing

We must consider how we want the system to behave when the job is retried. Is it going to re-refund customers that were already refunded? Is it going to send multiple emails? Is it going to loop over all conference attendees every time it retries?

To implement fault tolerance for our long-running job, we need to use a technique called **checkpointing**. It's saving a snapshot of the job's state so that it can be restarted from that point in case of failure.

```php
$this->attendees
	->skip(Cache::get('refunded', 0))
	->each(function($attendee) {
		$attendee->invoice->refund();
		Cache::increment('refunded');
		Mail::to($attendee)->send(...);
	});
```