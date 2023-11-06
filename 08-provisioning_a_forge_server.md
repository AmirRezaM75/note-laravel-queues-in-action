# Provisioning a Forge Server

Exceptions could be thrown due to a validation error sent from the server creation endpoint. Attempting the job five times is too much in this case.
For that reason, we will allow the job to be released five times but only allow it to fail—with a thrown exception—for two times.

```php
class ProvisionServer implements ShouldQueue
{
	public $tries = 5;
	public $maxExceptions = 2;

	public function handle()
	{
		if (! $this->server->forge_server_id) {
			$response = Http::timeout(5)->post(
				'.../servers', $this->payload
			)->throw()->json();
			$this->server->update([
				'forge_server_id' => $response['id']
			]);
			return $this->release(120);
		}

		$response = Http::timeout(5)->get(
			'.../servers/'.$this->server->forge_server_id,
		)->throw()->json();

		if ($response['status'] == 'provisioning') {
			return $this->release(60);
		}

		$this->server->update([
			'is_ready' => true
		]);
	}
}
```

## Handling Job Failure

We are going to send notification when the job fails.

```php
public function failed(Exception $e)
{
	Alert::create([
		'message' => "Provisioning failed!",
	]);
}
```