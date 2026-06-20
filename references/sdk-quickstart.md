# Rerout SDK quickstart (all languages)

Official SDKs wrap the REST API with typed models, error handling, retries, and
webhook signature verification. Source: <https://github.com/ModestNerds-Co/rerout-sdks>.
Current version: **0.4.0** (adds batch link creation and the `conversions`
namespace; 0.3.0 added `webhooks` management).

Every SDK exposes the same namespaces on the client: **links** (incl. batch),
**conversions**, **project**, **qr**, and **webhooks** (management), plus a
standalone signature **verifier**. Construct the client with the project API key
(`rrk_…`), ideally from `REROUT_API_KEY`.

The per-language snippets below focus on webhook management; the `links`,
`conversions`, batch, and QR methods follow the same `rerout.<namespace>.<method>`
shape. For example, in TypeScript:

```ts
const link = await rerout.links.create({ target_url: 'https://example.com/sale' })
const batch = await rerout.links.createBatch([
  { target_url: 'https://example.com/a' },
  { target_url: 'https://example.com/b' },
])
await rerout.conversions.record({ click_id: 'clk_…', event_name: 'purchase', value_cents: 4999, currency: 'USD' })
```

Python mirrors this as `rerout.links.create_batch([...])` and
`rerout.conversions.record(...)`; other languages use their idiomatic casing.

## TypeScript / JavaScript — `@rerout/sdk`
```bash
npm install @rerout/sdk
```
```ts
import { Rerout } from '@rerout/sdk'
const rerout = new Rerout({ apiKey: process.env.REROUT_API_KEY! })

const link = await rerout.links.create({ target_url: 'https://example.com/sale' })

const { endpoint, signing_secret } = await rerout.webhooks.create({
  name: 'Order events',
  url: 'https://example.com/hooks/rerout',
  events: ['link.created', 'link.clicked'],
})
const { endpoints, event_types } = await rerout.webhooks.list()
await rerout.webhooks.delete(endpoint.id)
```

## Python — `rerout`
```bash
pip install rerout
```
```python
from rerout import Rerout, CreateWebhookInput
rerout = Rerout(api_key="rrk_...")

created = rerout.webhooks.create(CreateWebhookInput(
    name="Order events",
    url="https://example.com/hooks/rerout",
    events=["link.created", "link.clicked"],
))
print(created.signing_secret)   # shown once
result = rerout.webhooks.list()
rerout.webhooks.delete(created.endpoint.id)
```

## Ruby — `rerout`
```ruby
# Gemfile: gem 'rerout'
rerout = Rerout::Client.new(api_key: ENV.fetch('REROUT_API_KEY'))
created = rerout.webhooks.create(
  Rerout::CreateWebhookInput.new(
    name: 'Order events',
    url: 'https://example.com/hooks/rerout',
    events: ['link.created', 'link.clicked'],
  ),
)
rerout.webhooks.list
rerout.webhooks.delete(created.endpoint.id)
```

## PHP — `rerout/sdk`
```bash
composer require rerout/sdk
```
```php
$rerout = new Rerout\Rerout(getenv('REROUT_API_KEY'));
$created = $rerout->webhooks()->create(new Rerout\Models\CreateWebhookInput(
    name: 'Order events',
    url: 'https://example.com/hooks/rerout',
    events: ['link.created', 'link.clicked'],
));
$rerout->webhooks()->list();
$rerout->webhooks()->delete($created->endpoint->id);
```

## Go — `github.com/ModestNerds-Co/rerout-sdks/go`
```bash
go get github.com/ModestNerds-Co/rerout-sdks/go@latest
```
```go
client, _ := rerout.NewClient(os.Getenv("REROUT_API_KEY"))
created, _ := client.Webhooks().Create(ctx, rerout.CreateWebhookInput{
    Name:   "Order events",
    URL:    "https://example.com/hooks/rerout",
    Events: []string{"link.created", "link.clicked"},
})
client.Webhooks().List(ctx)
client.Webhooks().Delete(ctx, created.Endpoint.ID)
```

## Java — `co.rerout:rerout-java:0.3.0`
```java
Rerout rerout = new Rerout(System.getenv("REROUT_API_KEY"));
CreatedWebhook created = rerout.webhooks().create(
    CreateWebhookInput.builder("Order events", "https://example.com/hooks/rerout")
        .events("link.created", "link.clicked").build());
rerout.webhooks().list();
rerout.webhooks().delete(created.getEndpoint().getId());
```

## Kotlin — `co.rerout:rerout-kotlin:0.3.0`
```kotlin
val rerout = Rerout(System.getenv("REROUT_API_KEY"))
val created = rerout.webhooks.create(CreateWebhookInput(
    name = "Order events",
    url = "https://example.com/hooks/rerout",
    events = listOf("link.created", "link.clicked"),
))
rerout.webhooks.list()
rerout.webhooks.delete(created.endpoint.id)
```

## .NET — `Rerout`
```bash
dotnet add package Rerout
```
```csharp
using var rerout = new ReroutClient(Environment.GetEnvironmentVariable("REROUT_API_KEY"));
var created = await rerout.Webhooks.CreateAsync(new CreateWebhookInput {
    Name = "Order events",
    Url = "https://example.com/hooks/rerout",
    Events = new[] { "link.created", "link.clicked" },
});
await rerout.Webhooks.ListAsync();
await rerout.Webhooks.DeleteAsync(created.Endpoint.Id);
```

## Dart / Flutter — `rerout`
```bash
dart pub add rerout   # or: flutter pub add rerout
```
```dart
final rerout = Rerout.initialize(apiKey: 'rrk_...');
final created = await rerout.webhooks.create(CreateWebhookRequest(
  name: 'Order events',
  url: 'https://example.com/hooks/rerout',
  events: ['link.created', 'link.clicked'],
));
await rerout.webhooks.list();
await rerout.webhooks.delete(created.endpoint.id);
```

## Rust — `rerout`
```bash
cargo add rerout
```
```rust
let rerout = Rerout::new(std::env::var("REROUT_API_KEY")?)?;
let created = rerout.webhooks().create(
    rerout::CreateWebhookInput::new(
        "Order events",
        "https://example.com/hooks/rerout",
        vec!["link.created".into(), "link.clicked".into()],
    ),
).await?;
rerout.webhooks().list().await?;
rerout.webhooks().delete(&created.endpoint.id).await?;
```

> Framework adapters (`Rerout.AspNetCore`, `rerout-spring-boot-starter`,
> `rerout/laravel`, `rerout-django`, `rerout-rails`) wrap the core SDK for
> **receiving** webhooks (DI wiring + verified controllers). For managing
> endpoints, use the core SDK's `webhooks` namespace shown above.
