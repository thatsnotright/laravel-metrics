# Laravel Metrics

[![Build](https://img.shields.io/scrutinizer/build/g/stechstudio/laravel-metrics.svg?style=flat-square)](https://scrutinizer-ci.com/g/stechstudio/laravel-metrics)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)
[![Quality Score](https://img.shields.io/scrutinizer/g/stechstudio/laravel-metrics.svg?style=flat-square)](https://scrutinizer-ci.com/g/stechstudio/laravel-metrics)
[![Total Downloads](https://img.shields.io/packagist/dt/stechstudio/laravel-metrics.svg?style=flat-square)](https://packagist.org/packages/stechstudio/laravel-metrics)

This package makes it incredibly easy to ship app metrics to backends such as InfluxDB or CloudWatch.

There are two major components: a facade that lets you create metrics on your own, and an event listener to automatically send metrics for Laravel events.
   
## Installation

You know the drill...

```
composer require stechstudio/laravel-metrics
```

If you're running Laravel 5.5, you're done. For Laravel 5.4 and earlier, add the following service provider and facade to `config/app.php`:

```php
'providers' => [
    ...
    STS\Metrics\MetricsServiceProvider::class,
],

'aliases' => [
    ..
    'Metrics' => STS\Metrics\MetricsFacade::class,
],
```

## Backend configuration

### InfluxDB

Add the following to your `.env` file:

```
METRICS_BACKEND=influxdb
IDB_USERNAME=...
IDB_PASSWORD=...
IDB_HOST=...
IDB_DATABASE=...

# Only if you are not using the default 8086
IDB_TCP_PORT=...

# If you want to send metrics over UDP instead of TCP
IDB_UDP_PORT=...
```

### CloudWatch

First make sure you have AWS itself properly setup. That means installing [`aws-sdk-php-laravel`](https://github.com/aws/aws-sdk-php-laravel) and following the configuration instructions for that package.
 
From there, you simply need to add:

```
METRICS_BACKEND=cloudwatch
CLOUDWATCH_NAMESPACE=...
```

### NullDriver (for development)

If you need to disable metrics just set the backend to null:

```
METRICS_BACKEND=null
```

This `null` driver will simply discard any metrics. 

## Sending an individual metric

You can create metric by using the facade like this:

```php
Metrics::create('order_placed')
    ->setValue(1)
    ->setTags([
        'source' => 'email-campaign',
        'user' => 54
    ]);
```

The only required attribute is the `name`, everything else is optional. 

## Driver mapping

This is how we are mapping metric attributes in our backends.  

| Metric attribute | InfluxDB      | CloudWatch        |
| ---------------- | ------------- | ----------------- |
| name             | measurement   | MetricName        |
| value            | fields[value] | Value             |
| unit             | _ignored_     | Unit              |
| resolution       | _ignored_     | StorageResolution |
| tags             | tags          | Dimensions        |
| extra            | fields        | _ignored_         |
| timestamp        | timestamp     | Timestamp         |

See the [CloudWatch docs](http://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_MetricDatum.html) and [InfluxDB docs](https://docs.influxdata.com/influxdb/latest/concepts/key_concepts/) for more information on their respective data formats. Note we only do minimal validation, you are expected to know what data types and formats your backend supports for a given metric attribute.

## Sending metrics from Laravel events

The main motivation for this library was to send metrics automatically when certain events occur in a Laravel application. So this is where things really get fun! 

Let's say you have a simple Laravel event called OrderReceived:

```php
class OrderReceived {
    protected $order;
    
    public function __construct($order)
    {
        $this->order = $order;
    }
}
```

The first step is to implement an interface:
 
```php
use STS\Metrics\Contracts\ShouldReportMetric;

class OrderReceived implements ShouldReportMetric {
```

This will tell the global event listener to send a metric for this event. 

There are two different ways you can then provide the metric details.

### 1. Use the `ProvidesMetric` trait

You can also include a trait that helps with building this metric:
 
```php
use STS\Metrics\Contracts\ShouldReportMetric;
use STS\Metrics\Traits\ProvidesMetric;

class OrderReceived implements ShouldReportMetric {
    use ProvidesMetric;
```

In this case, the trait will build a metric called `order_received` (taken from the class name) with a value of `1`.

#### Customizing event metric data

If you decide to use the trait, you likely will want to customize the event metric data. 

You can provide metric data with class attributes:

```php
class OrderReceived implements ShouldReportMetric {
    use ProvidesMetric;
    
    protected $metricName = "new_order";
    protected $metricTags = ["category" => "revenue"];
    ...
```

Or if some of your metric data is dynamic you can use getter methods:

```php
public function getMetricValue()
{
    return $this->order->total;
}
```

You can provide any of our metric attributes using these class attributes or getter methods. 

### 2. Create the metric yourself

Depending on how much detail you need to provide for your metric, it may be simpler to just build it yourself. In this case you can ditch the trait and simply provide a public `createMetric` function that returns a new `Metric` instance:

```php
use STS\Metrics\Contracts\ShouldReportMetric;
use STS\Metrics\Metric;

class OrderReceived implements ShouldReportMetric {
    protected $order;
    
    public function __construct($order)
    {
        $this->order = $order;
    }
    
    public function createMetric()
    {
        return (new Metric('order_received'))
            ->setValue(...)
            ->setTags([...])
            ->setTimestamp(...)
            ->setResolutions(...);
    }
}
```
