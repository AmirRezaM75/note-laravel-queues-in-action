# Preventing a Double Refund

The `CancelConference` job from the last challenge may run for as long as 5 hours to refund all the attendees and send them an email. While doing so, the worker won't be able to pick up any other job since it can only process one job at a time. Moreover, if the worker crashes
at any point, Laravel will wait for 18060 seconds (5 hours and 1 minute) to retry the job.
It would be better to dispatch multiple shorter jobs instead of a long-running one.
```php
$this->attendees
->each(function($attendee) {
	RefundAttendee::dispatch($attendee);
});
```

It's always a good practice to avoid any negative side effects from the same job instance running multiple times. A fancy name for this is **Idempotence**.

```php
public function handle()
{
	if ($this->attendee->invoice->refunded){
		return;
	}
	$this->attendee->invoice->refund();
	$this->attendee->invoice->update([
		'refunded' => true
	]);
	Mail::to($this->attendee)->send(...);
}
```

When we call refund on an invoice, our code will send an HTTP request to the billing
provider to issue a refund. If the refund was issued but a response wasn't sent back due to a
networking issue, our request will error, and the invoice will never be marked as refunded.

In this situation, our billing provider is the source of truth. Only they know—for sure—if the
invoice was refunded.

```php
public function handle()
{
if ($this->attendee->invoice->wasRefunded()) {
return;
}
$this->attendee->invoice->refund();
Mail::to($this->attendee)->send(...);
}
```