# Retrying Failed Jobs

Here's what Laravel does by default when a job
fails:
1. Deletes the job from the queue.
2. Reports the failure in the terminal window running the queue:work command.
3. Dispatches a `JobFailed` event.
4. Stores the job information in a `failed_jobs` table.

In message queueing a **dead letter queue (DLQ)** is a service implementation to store messages that the messaging system cannot or should not deliver.

## Inspecting the Dead-letter Queue

```
php artisan queue:failed
```

This command outputs a table in the terminal window that contains all the failed jobs records.

## Retrying a Job Manually

Once the issue is fixed, developers will want to send all those failed jobs:

```
php artisan queue:retry 67b31b39-26df-4b39-b2db-c4fb78088fd1
```

Here are the steps Laravel takes to retry a job:
1. Queries the `failed_jobs` table.
2. Gets the record.
3. Dispatches the job to the queue.
4. Deletes the record.

Laravel allows us to retry all of the failed jobs by running a single command:

```
php artisan queue:retry all
```

However, for some cases where the job payload is calculated incorrectly, a retry won't help. In such situations, Laravel provides two commands:

```
php artisan queue:prune-failed
```

it will delete failed jobs older than **24 hours**. We can control the
number of hours to retain failed jobs by providing a `hours` option:

The other command deletes all failed jobs:

```
php artisan queue:flush
```