
# Sending Webhooks

The challenge we have here is implementing an exponential webhook retry policy in which the retry backoff is increased by 15 minutes after every attempt. The service keeps attempting the webhook for 24 hours and sends an alert to the partner at the end of this period.

```php
class SendWebhook implements ShouldQueue
{
	public $tries = 15;
	public $backoff = [15, 30, 45, 60, 75, 90, ...];

	public function handle() {
		Http::post(...)->throw()
	}
}
```

Setting the backoff might be burdensome in this situation. We also could do the math and calculate how many attempts a whole day can allow based on our exponential backoff. But there's an easier way:

```php
class SendWebhook implements ShouldQueue
{
	// unlimited number of times
	public $tries = 0;
	
	public function handle() {
		$response = Http::post(...);

		if ($response->failed()) {
			$this->release(
				now()->addMinutes(15 * $this->attempts())
			);
		}
	}

	public function retryUntil()
	{
		return now()->addDay();
	}

	public function failed(Exception $e)
	{
		Mail::to(
			$this->integration->email
		)->send(...);
	}
}
```

> The expiration is calculated by calling the `retryUntil()` method when the job is first dispatched. If you release the job back, the expiration will not be re-calculated.

Now the worker will fail this job if it was attempted after the 24-hour period.

When a job exceeds the assigned number of attempts or reaches the expiration time, the worker will call this `failed()` method and pass a `MaxAttemptsExceededException` exception.