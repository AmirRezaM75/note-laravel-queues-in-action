# High Priority Jobs

But even with multiple workers running, if the queue is filled with
`SendVerificationMessage` jobs and we push a `ProcessPayment` job, the workers won't pick that job up until all previous `SendVerificationMessage` jobs are processed.

```php
ProcessPayment::dispatch($payment)->onQueue('payments');
```

or

```php
class ProcessPayment implements ShouldQueue
{
	public $queue = 'payments';
}
```

## Processing Jobs from Multiple Queues

```
php artisan queue:work --queue=payments,default
```

This will instruct the worker to dequeue jobs from the payments and default queues, with the payments queue having a higher priority.
Jobs from the default queue will not be processed until the payments queue is cleared.

## Using Separate Workers for Different Queues

If the payments queue is always filled with jobs, jobs from queues with lower priority may not run at all.

```
php artisan queue:work --queue=payments
php artisan queue:work --queue=default
```

One thing with this, though, is if the default queue is empty while the payments queue is full, the second worker will stay idle.

```
php artisan queue:work --queue=payments,default
php artisan queue:work --queue=default,payments
```
