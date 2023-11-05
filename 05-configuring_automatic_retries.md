# Configuring Automatic Retries

This worker will attempt every job 3 times before considering it a failure.

```
php artisan queue:work --tries=3
```

Laravel will release it back to the queue so workers can pick it up again. You can think of **releasing** a job back to the queue as **pushing** a new job.
There's no guarantee that a job will be retried immediately nor the same worker will retry it.

## Job-Specific Tries

```php
class SendVerificationMessage implements ShouldQueue
{
	public $tries = 3;
}
```

Whatever value we use in the `$tries` public property will override the value specified in the `--tries` worker option.

## Delaying Tries

In many cases, we may want to control the interval between retries to give the service enough time to recover from a transient failure.

```php
class SendVerificationMessage implements ShouldQueue
{
	public $tries = 3;
	public $backoff = 20;
}
```

Keep in mind that the `backoff` time only makes the job invisible to workers during the interval, it doesn't guarantee the job will be retried exactly after the `backoff` period passes.

## Exponential Backoff

```php
class SendVerificationMessage implements ShouldQueue
{
	public $tries = 4;
	public $backoff = [1, 5, 10];
}
```

In this example, the retry delay will be 1 second for the first retry, 5 seconds for the second retry, 10 seconds for the third retry, and 10 seconds for every subsequent retry if there are more attempts remaining.

We can also configure backoff on a worker level:

```
php artisan queue:work --backoff=20,60
```
