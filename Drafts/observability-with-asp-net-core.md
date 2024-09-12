It is crucial to understand an application's behavior, either to analyze specific events to find errors, monitor the application's performance, or track activities in a distributed system. Observability is the capability of an application to do that, through logs, metrics, and tracing.

In this article I’ll be talking about metrics, how to configure an ASP NET Core application to generate metrics, set up Prometheus and Grafana to collect and visualize the metrics.

Metrics are the application's telemetry data over time. For example, the number of requests of an API, the number of errors, and the latency of the request. It is essential to collect that data and monitor it, for example to react to application degradation incidents or understand the application behavior to implement performance improvements.

### Creating an API

The first step here is to create an API using ASP NET Core. I created an API for a fictitious sporting goods store called ShopSports. It has an `/api/Products` endpoint that returns a list of products, containing the name, price and availability. This API needs to communicate with two external services, one for retrieving the products prices, and another to get the procuts availability. In order to simulate a real use scenario, this API generates random delays during the products query to the database, and delays when retrieving data in the other services.

```csharp
public async Task<IActionResult> GetAsync()
{
    var products = await _productRepository.GetAsync();
    var fillPricesTask = _priceService.FillPricesAsync(products);
    var fillAvailabilityTask = _inventoryService.FillAvailabilityAsync(products);
    await Task.WhenAll(fillPricesTask, fillAvailabilityTask);
    return Ok(products);
}
```

### Open Telemetry

