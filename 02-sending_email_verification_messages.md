# Configuring the Queue

We're going to use the database queue driver. It's easy to set up and gives us a high level of visibility.

```
QUEUE_CONNECTION=database
```

```
php artisan queue:table
```

Running this command will create a migration file.

The `queue` column will hold the name of the queue a job is stored in.

The `payload` column will hold the body of the job, which is the string representation of the job object.

The `attempts` column will hold the number of times the job has been attempted.

The `reserved_at` column will hold the timestamp of when a worker picked up this job.

The `available_at` column will hold the timestamp of when workers are allowed to pick this job up.

The `created_at` column will hold the timestamp of when the job was dispatched to the queue.

# Sending Email Verification Messages

Create a job to send email verification notification:

```
php artisan make:job SendVerificationMessage
```

```php
namespace App\Jobs;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
class SendVerificationMessage implements ShouldQueue
{
	use Dispatchable;
	use InteractsWithQueue;
	use Queueable;
	use SerializesModels;
	
	public $user;
	
	public function __construct(User $user)
	{
		$this->user = $user;
	}
	
	public function handle()
	{
		Mail::to($this->user)->send(...);
	}
}
```

Arguments passed to the static `dispatch()` method will be
transferred to the job instance automatically.

```php
SendVerificationMessage::dispatch($user);
```

> Starting Laravel 8.0, you can use `__invoke` instead of `handle` for the method name.

Each worker can only process a single job at any given time. To process multiple jobs in parallel, we'll need to start more workers:

```
php artisan queue:work
php artisan queue:work
```

You may think that since the two workers are running in parallel, then there's a chance that both of them will pick up the same job resulting in sending the same verification email to the same user twice. But that's not going to happen. All queue storage drivers use **atomic locks** to prevent multiple workers from picking the same job.

```sql
SELECT * FROM `jobs`
WHERE ...
ORDER BY `id` ASC
FOR UPDATE SKIP LOCKED
LIMIT 1
```

The last line in the query has the `FOR UPDATE` instruction, which tells MySQL to take an exclusive lock on the returned row. It also has `SKIP LOCKED`, which tells MySQL to skip any locked row and exclude it from the query.