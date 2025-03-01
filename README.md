<p align="center"><img src="https://cdn.growthbook.io/growthbook-logo@2x.png" width="400px" /></p>

# Growth Book - PHP

Powerful Feature flagging and A/B testing for PHP.

![Build Status](https://github.com/growthbook/growthbook-php/workflows/Build/badge.svg)

-  **No external dependencies**
-  **Lightweight and fast**
-  **No HTTP requests** everything is defined and evaluated locally
-  **PHP 7.1+** with 100% test coverage and phpstan on the highest level
-  **Advanced user and page targeting**
-  **Use your existing event tracking** (GA, Segment, Mixpanel, custom)
-  **Adjust variation weights and targeting** without deploying new code

## Installation

Growth Book is available on Composer:

`composer require growthbook/growthbook`

## Quick Usage

```php
// Get current feature flags from GrowthBook API
// TODO: persist the JSON in a database or cache in production
const FEATURES_ENDPOINT = 'https://cdn.growthbook.io/api/features/key_prod_abc123';
$apiResponse = json_decode(file_get_contents(FEATURES_ENDPOINT), true);
$features = $apiResponse["features"];

// Create a GrowthBook instance
$growthbook = Growthbook\Growthbook::create()
  ->withFeatures($features)
  ->withAttributes([
    'id' => $userId,
    'someCustomAttribute' => true
  ]);

// Feature gating
if ($growthbook->isOn("my-feature")) {
  echo "It's on!";
} else {
  echo "It's off :(";
}

// Remote configuration with fallback
$color = $growthbook->getValue("button-color", "blue");

echo "<button style='color:${color}'>Click Me!</button>";
```

Some of the feature flags you evaluate might be running an A/B test behind the scenes which you'll want to track in your analytics system.

At the end of the request, you can loop through all experiments and track them however you want to:

```php
$impressions = $growthbook->getViewedExperiments();
foreach($impressions as $impression) {
  // Whatever you use for event tracking
  Segment::track([
    "userId" => $userId,
    "event" => "Experiment Viewed",
    "properties" => [
      "experimentId" => $impression->experiment->key,
      "variationId" => $impression->result->variationId
    ]
  ]);
}
```

## The Growthbook Class



The `Growthbook` class has a number of properties.  These can be set using a Fluent interface or can be passed into a constructor using an associative array. Every property also has a getter method if needed. Here's an example:

```php
// Using the fluent interface
$growthbook = Growthbook\Growthbook::create()
  ->withFeatures($features)
  ->withAttributes($attributes);

// Using the constructor
$growthbook = new Growthbook\Growthbook([
  'features' => $features,
  'attributes' => $attributes
]);

// Getter methods
print_r($growthbook->getFeatures());
print_r($growthbook->getAttributes());
```

Note: you can also use the fluent methods (e.g. `withFeatures`) at any point to update properties.

### Features

Defines all of the available features plus rules for how to assign values to users. Features are usually fetched from the GrowthBook API and persisted in cache or a database in production.

Feature definitions are defined in a JSON format. You can fetch them directly from the GrowthBook API:

```php
const FEATURES_ENDPOINT = 'https://cdn.growthbook.io/api/features/key_prod_abc123';
$apiResponse = json_decode(file_get_contents(FEATURES_ENDPOINT), true);
$features = $apiResponse["features"];
```

Or, you can use a copy stored in your database or cache server instead:

```php
// From database/cache
$jsonString = '{"feature-1": {...}, "feature-2": {...}}';
$features = json_decode($jsonString, true);
```

We recomend the cache approach for production.

### Attributes

You can specify attributes about the current user and request. These are used for two things:

1.  Feature targeting (e.g. paid users get one value, free users get another)
2.  Assigning persistent variations in A/B tests (e.g. user id "123" always gets variation B)

Attributes can be any JSON data type - boolean, integer, float, string, or array.

```php
$attributes = [
  'id' => "123",
  'loggedIn' => true,
  'deviceId' => "abc123def456",
  'age' => 21,
  'tags' => ["tag1", "tag2"],
  'account' => [
    'age' => 90
  ]
];
```

If you want to update attributes later, please note that the `withAttributes` method completely overwrites the attributes object.  You can use `array_merge` if you only want to update a subset of fields:

```ts
// Only update the url attribute
$growthbook->withAttributes(array_merge(
  $growthbook->getAttributes(),
  [
    'url' => '/checkout'
  ]
));
```

### Tracking Experiments

Any time an experiment is run to determine the value of a feature, you want to track that event in your analytics system.

You can either do this via a callback function:

```php
$trackingCallback = function (
  Growthbook\InlineExperiment $experiment, 
  Growthbook\ExperimentResult $result
) {  
  // Segment.io example
  Segment::track([
    "userId" => $userId,
    "event" => "Experiment Viewed",
    "properties" => [
      "experimentId" => $experiment->key,
      "variationId" => $result->variationId
    ]
  ]);
};

// Fluent interface
$growthbook = Growthbook\Growthbook::create()
  ->withTrackingCallback($callback);

// Using the construtor
$growthbook = new Growthbook([
  'trackingCallback' => $trackingCallback
]);

// Getter method
$trackingCallback = $growthbook->getTrackingCallback();
```

Or track all events at the end of the request by looping through an array:

```php
$impressions = $growthbook->getViewedExperiments();
foreach($impressions as $impression) {
  // Segment.io example
  Segment::track([
    "userId" => $userId,
    "event" => "Experiment Viewed",
    "properties" => [
      "experimentId" => $impression->experiment->key,
      "variationId" => $impression->result->variationId
    ]
  ]);
}
```

Or, you can pass the impressions onto your front-end and fire analytics events from there. To do this, simply add a block to your template (shown here in plain PHP, but similar idea for Twig, Blade, etc.).

```php
<script>
<?php foreach($growthbook->getViewedExperiments() as $impression): ?>
  // tracking code goes here
<?php endforeach; ?>
</script>
```

Below are examples for a few popular front-end tracking libraries:

#### Google Analytics
```php
ga('send', 'event', 'experiment', 
  "<?= $impression->experiment->key ?>", 
  "<?= $impression->result->variationId ?>", 
  {
    // Custom dimension for easier analysis
    'dimension1': "<?= 
      $impression->experiment->key.':'.$impression->result->variationId 
    ?>"
  }
);
```

#### Segment
```php
analytics.track("Experiment Viewed", <?=json_encode([
  "experimentId" => $impression->experiment->key,
  "variationId" => $impression->result->variationId 
])?>);
```

#### Mixpanel
```php
mixpanel.track("Experiment Viewed", <?=json_encode([
  'Experiment name' => $impression->experiment->key,
  'Variant name' => $impression->result->variationId 
])?>);
```

## Using Features

There are 3 main methods for interacting with features.

- `$growthbook->isOn("feature-key")` returns true if the feature is on
- `$growthbook->isOff("feature-key")` returns false if the feature is on
- `$growthbook->getValue("feature-key", "default")` returns the value of the feature with a fallback


In addition, you can use `$growthbook->getFeature("feature-key")` to get back a `FeatureResult` object with the following properties:

- **value** - The JSON-decoded value of the feature (or `null` if not defined)
- **on** and **off** - The JSON-decoded value cast to booleans
- **source** - Why the value was assigned to the user. One of `unknownFeature`, `defaultValue`, `force`, or `experiment`
- **experiment** - Information about the experiment (if any) which was used to assign the value to the user
- **experimentResult** - The result of the experiment (if any) which was used to assign the value to the user

## Inline Experiments

Instead of declaring all features up-front and referencing them by ids in your code, you can also just run an experiment directly. This is done with the `$growthbook->runInlineExperiment` method:

```js
$exp = Growthbook\InlineExperiment::create(
  "my-experiment", 
  ["red", "blue", "green"]
);

// Either "red", "blue", or "green"
echo $growthbook->runInlineExperiment($exp)->value; 
```

As you can see, there are 2 required parameters for experiments, a string key, and an array of variations.  Variations can be any data type, not just strings.

There are a number of additional settings to control the experiment behavior. The methods are all chainable. Here's an example that shows all of the possible settings:

```php
$exp = Growthbook\InlineExperiment::create("my-experiment", ["red","blue"])
  // Run a 40/60 experiment instead of the default even split (50/50)
  ->withWeights([0.4, 0.6])
  // Only include 20% of users in the experiment
  ->withCoverage(0.2)
  // Targeting conditions using a MongoDB-like syntax
  ->withCondition([
    'country' => 'US',
    'browser' => [
      '$in' => ['chrome', 'firefox']
    ]
  ])
  // Use an alternate attribute for assigning variations (default is 'id')
  ->withHashAttribute("sessionId")
  // Namespaces are used to run mutually exclusive experiments
  // Another experiment in the "pricing" namespace with a non-overlapping range
  //   will be mutually exclusive (e.g. [0.5, 1])
  ->withNamespace("pricing", 0, 0.5);
```

### Inline Experiment Return Value

A call to `runInlineExperiment` returns an `ExperimentResult` object with a few useful properties:

```php
$result = $growthbook->runInlineExperiment($exp);

// If user is part of the experiment
echo($result->inExperiment); // true or false

// The index of the assigned variation
echo($result->variationId); // e.g. 0 or 1

// The value of the assigned variation
echo($result->value); // e.g. "A" or "B"

// The user attribute used to assign a variation
echo($result->hashAttribute); // "id"

// The value of that attribute
echo($result->hashValue); // e.g. "123"
```

The `inExperiment` flag is only set to true if the user was randomly assigned a variation. If the user failed any targeting rules or was forced into a specific variation, this flag will be false.
