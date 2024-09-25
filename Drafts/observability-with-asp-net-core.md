It is crucial to understand an application's behavior, either to analyze specific events to find errors, monitor the application's performance, or track activities in a distributed system. Observability is the capability of an application to do that, through logs, metrics, and tracing.

In this article I’ll be talking about metrics, how important and powerful they can be, and showing how they can be used, showing its usage in a real scenario. For that, we will be configuring an ASP NET Core application to generate metrics and set up Prometheus and Grafana to collect and visualize the metrics.

Metrics are the application's telemetry data over time. For example, the number of requests of an API, the number of errors, and the latency of the request. It is essential to collect that data and monitor it, for example, to react to application degradation incidents or understand the application behavior to implement performance improvements.

### Creating an API

The first step here is to create an API using ASP NET Core. I made an API for a fictitious sporting goods store called ShopSports. It has an `/api/Products` endpoint that returns a list of products, containing the name, price and availability. This API needs to communicate with two external services, one for retrieving the prices of the products, and another to get the procuts's availability.

To simulate a real-use scenario, this API generates random delays during the products query to the database in the `ProductRepository`, and delays when retrieving data in `PriceService` and `InventoryService`. This is the products API implementation:

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

The next step is to configure the application to generate metrics. Open Telemetry will be used for that, which is an open-source platform for observability, it is technology agnostic, supports logs, metrics and distributed tracings. ASP NET Core has its Open Telemetry implementation, we add it by installing the following packages:

```csharp
OpenTelemetry.Instrumentation.AspNetCore
OpenTelemetry.Extensions.Hosting
OpenTelemetry.Exporter.Console
```

After installing, we add the following configuration to the dependency injection container configuration:

```csharp
var openTelemetryBuilder = builder.Services.AddOpenTelemetry();

openTelemetryBuilder.ConfigureResource(resource => resource
    .AddService(builder.Environment.ApplicationName));

openTelemetryBuilder.WithMetrics(metrics => metrics
    .AddAspNetCoreInstrumentation()
    .AddConsoleExporter());
```

This configuration adds some metrics by default. We will be looking at the one called `http.server.request.duration`. As the name suggests, this metric measures the request duration in seconds.

As we have added the `AddConsoleExporter` to the configuration, a console exporter will be available. An exporter is a way of accessing the snapshot of the metrics at a given time. The console exporter will output the metrics from time to time. This is the console output after making three request to the `/api/Producs` endpoint:

![Console exporter](https://raw.githubusercontent.com/sylvester-henrique/hashnode-articles/main/Drafts/http-server-request-duration-console.png)

I highlight three points:

- The type of this metric is Histogram, which registers the frequency of values within a range.
- Note that we have defined ranges of values and a number for each range. These ranges of values are called buckets. In this example, two of these requests took between 0.75 and 1 second, and the other took between 1.2 and 5 seconds.
- We have additional information attached to the metrics, for example, the `http.response.status_code` and `http.route`. These information are called tags, and can be useful for filtering data.

### Prometheus

Now that the application is instrumenting the metrics, we need a system to collect them. We will be using Prometheus, which uses a pulling system to collect the metrics of an application. The process of collecting metrics is called scrapping. In order for Prometheus to scrap the metrics it is necessary to configure an exporter, that will provide an endpoint for the scrapping. That said, we will install the package to configure the exporter endpoint:

```csharp
OpenTelemetry.Exporter.Prometheus.AspNetCore
```

Add the `AddPrometheusExporter` call to the dependency injection container:

```csharp
openTelemetryBuilder.WithMetrics(metrics => metrics
    .AddAspNetCoreInstrumentation()
    .AddConsoleExporter()
    .AddPrometheusExporter());
```

And the `MapPrometheusScrapingEndpoint` call somewhere after the `Build` call:

```csharp
var app = builder.Build();
...
// Configure the Prometheus scraping endpoint
app.MapPrometheusScrapingEndpoint();
```

Now, the metrics will be available at `/metrics` endpoint.

The next step is to install Prometheus. After the installation, it is necessary to specify the target for the scrapping endpoint in the Prometheus YAML file. The target is `localhost:5068`, which is the URL that ShopSports API is running. Besides, for testing purposes, I defined the scrape interval of 5s. That means that Prometheus will do the pulling at every 5 seconds:

```yaml
scrape_interval: 5s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:5068"]
```

After running Prometheus, access its default endpoint at `http://localhost:9090`. On the Prometheus interface, it is possible to query metrics using the PromQL language. Also, we can view a graphic for the metric over time.

For exemplification, I'll be sending a request to the `/api/Products` endpoint, every 3 seconds, using Postman. Meanwhile, I'll be filtering the request duration average in Prometheus for that endpoint, using the PromQL query:

```
increase(http_server_request_duration_seconds_sum{http_route='api/Products',http_response_status_code='200'}[5m])
/
increase(http_server_request_duration_seconds_count{http_route='api/Products',http_response_status_code='200'}[5m])
```

First, the query calculates the increase of the sum of all request durations in an interval of 5 minutes. Then, it calculates the increase in the last 5 minutes, of count of all requests. Finally, it divides those values to get the average request duration in the informed interval. This is the resultant Prometheus graph:

![Prometheus graph](https://raw.githubusercontent.com/sylvester-henrique/hashnode-articles/main/Drafts/http-server-request-duration-prometheus.png)

Note that at `21:10:50` the request duration average was ≈ `1.62` seconds.

### Grafana

Although it is possible to build graphs for our metrics using Prometheus, as seen previously, we also can use Grafana for that. Besides, Grafana has a more robust system for creating monitoring dashboards, alerts, and has a richer user interface.

After installing Grafana, we need to configure a datasource in which it will be getting the monitoring data. In Grafana's datasource configuration choose Prometheus, and specify our Prometheus endpoint which is `http://localhost:9090`.

Now, we'll be creating a dashboard for the request duration metric. One way of viewing samples of request duration over time is using the `histogram_quantile` function. The first parameter for this function is the percentile. For example, when the percentile 95 shows 2 seconds, this indicates that 95% of the requests took below 2 seconds, while only 5% took 2 seconds or more. For example, this is the query for percentile 95 (P95):

```
histogram_quantile(0.95, sum by(le) (increase(http_server_request_duration_seconds_bucket{http_route="api/Products", http_response_status_code="200"}[5m])))
```

In a single Grafana dashboard, we will specify three queries similar to that, one for percentile 90 (P90), and other two for P95 and P99:

![Grafana http server request duration](https://raw.githubusercontent.com/sylvester-henrique/hashnode-articles/main/Drafts/http-server-request-duration-grafana.png)

As we can see in the yellow line, at `18:14:15` 95% of the request durations took less than `2.41` seconds. This dashboard is very useful to give an overview of how most of the clients of the API are experiencing it in terms of request duration.

### Custom metrics

In the first example, we used a default metric generated by the Open Telemetry implementation for ASP NET Core. Besides that, it is possible to create custom metrics to instrument specific events in our applications. For example, a metric to count errors in specific scenarios.

In the following example, we will modify the code to generate random errors in three scenarios:

- When querying products in the database using `ProductRepository`
- When getting prices in the `PriceService`
- When retrieving availabilities in the `InventoryService`

We will need a helper class to generate random errors:

```csharp
public static class ErrorGenerator
{
    public static void Execute(int probability, string message)
    {
        var hasError = new Random().Next(0, 100) < probability;
        if (hasError)
        {
            throw new Exception(message);
        }
    }
}
```

And a custom metric class for the `/api/Products` endpoint. Also, an enum for the error types:

```csharp
public class ProductsMetrics : IProductsMetrics
{
    private const string NAME = "Get.Products";
    private readonly Counter<long> _getProductsErrorCounter;

    public ProductsMetrics()
    {
        var getProdutcsErrorMeter = new Meter(NAME);
        _getProductsErrorCounter = getProdutcsErrorMeter.CreateCounter<long>("get.products.error.count");
    }

    public static string Name => NAME;

    public void RegisterError(GetProductError getProductError)
    {
        var tags = new KeyValuePair<string, object?>(nameof(getProductError), getProductError.ToString());
        _getProductsErrorCounter.Add(1, tags);
    }
}

public enum GetProductError
{
    QueryProducts,
    FillPrices,
    FillAvailability,
}
```

The name of the metric is `get.products.error.count`, which is a counter, and therefore will register the error count over time. Note that the metric has a tag attached to it. This tag is called `getProductError` and will help identify which error occurred.

This is the code for `FillAvailabilityAsync`, that when an error occurs, will register the `get.products.error.count` metric.

```csharp
public async Task FillAvailabilityAsync(IEnumerable<Product> products)
{
    try
    {
        await Delayer.ExecuteAsync(500);
        ErrorGenerator.Execute(3, "Failed to fill products availability.");
        foreach (var product in products)
        {
            product.IsAvailable = _random.Next(2) == 1;
        }
    }
    catch (Exception)
    {
        // Register the error metric
        _productsMetrics.RegisterError(GetProductError.FillAvailability);
        throw;
    }
}
```

Finally, add the metric to the `openTelemetryBuilder` using the `AddMeter` method:

```csharp
var customMetricsNames = new string[] { ProductsMetrics.Name };

openTelemetryBuilder.WithMetrics(metrics => metrics
    .AddMeter(customMetricsNames)
    .AddAspNetCoreInstrumentation()
    .AddConsoleExporter()
    .AddPrometheusExporter());
```

The next step is to create a query and a visualization for this metric. This is the query for the error count, that is filtering by `FillPrices` error:

```
increase(get_products_error_count_total{getProductError="FillPrices"}[5m])
```

At the Grafana dashboard, we can visualize error counts for the three specified errors:

![Grafana get products error count](https://raw.githubusercontent.com/sylvester-henrique/hashnode-articles/main/Drafts/get-products-error-count-grafana.png)

We can see that at `18:04:00` the count for `QueryProducts` error was ≈ 3.33, the count for `FillPrices` error was ≈ 2, while there was no `FillAvailability` errors. 

### Conclusion

We have seen the request duration metric, which later can be very useful for querying request duration averages, and visualize request counts for multiple endpoints. This information will help us to identify which endpoints have the high average durations, and which endpoints have the higher number of requests. Combining these two information we can take action to optimize the performance of critical endpoints with high numbers of requests that also have high average request duration. 

We have also seen the error count metric, which is a custom metric for the application. With this information, we could take actions to reduce specific errors that are occurring the most, to increase the resilience of the application.

Another scenario is when degradation incidents occur. Those metrics can help us to identify which point of the application is responsible for the degradation. In addition, Grafana provides a way to alarm when a metric surpasses a determined threshold, which can indicate that an application will begin to degrade soon.

The code shown here can be found at this repository: [asp-net-core-metrics-example-1](https://github.com/sylvester-henrique/asp-net-core-metrics-example-1)
