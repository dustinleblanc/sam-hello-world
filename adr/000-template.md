# Template - Do Not Remove

## Context

Note: More details about the applications that require this update are needed to reason about the right solution. Implementation is going to vary greatly depending on how the applications control their dependencies, what DevOps processes are in place, what canonical tools the team is already using to manage operations, etc. The description is intentionally vague to reflect this.

This ADR proposes a solution to keep the `package` dependency of our apps updated on a quarterly schedule by fetching new release information, and pulling the code to our own S3 bucket so that we can distribute the software ourselves.

## Decision

Utilize a cron job that executes a script that hits the third party's update rss feed. This script will first check if there are any releases that we have no already tracked and fetched. It will do this by referencing our internal release record (updated on every new release that is pulled, could be either a database record or perhaps even a flatfile).

If there is a new release, we'll update our internal record and fetch the new release (using an appropriate http client for the language used, such as Guzzle for PHP).

```php




```
Once we have a copy, we'll upload the tar file to our S3 bucket, and record the URL for the package in our release record, along with it's version, the description, and the date.

We'll then expose our own feed to our internal applications, in the most convenient format for the consumers (XML, JSON, etc).

Depending on the infrastructure and tools available, either the individual apps will query our feed and pull the new version, perhaps via their own cron jobs. If this is not possible, given we can provide source control access to this new tool, an automation script could be written to check out the source of each application, update it's dependencies to the new version, and then submit a pull request to the repository for review, or automatically merge and commit the results if risk is deemed low.

It is presumed each application is being deployed via CI/CD pipeline, so once the merge is handled, the application would be deployed.

A rough sketch of the script that would check for and download/upload the releases could look like this in PHP:

```php
<?php

// Include the Guzzle library
require 'vendor/autoload.php';

// Use the Guzzle HTTP client
use GuzzleHttp\Client;

// Create a new Guzzle client
$client = new Client();

// Set the RSS feed URL
$rss_url = 'http://example.com/package/rss';

// Send a request to the RSS feed URL using Guzzle
$response = $client->request('GET', $rss_url);

// Parse the response body as XML
$xml = $response->xml();

// Loop through the <item> elements in the XML
foreach ($xml->channel->item as $item) {
  // Get the package version from the <title> element
  $version = (string) $item->title;

  // Check if the package version is new
  if (isNewVersion($version)) {
    // Download the package
    downloadPackage($item->link, $version);

    $newUrl = uploadToS3($version);
    // Record the new version in the database
    recordVersion($version, $url, $item->desc);
  }
}

function downloadPackage($url, $version) {
  $resource = fopen("/downloads/package/{$version}", 'w');
  $stream = GuzzleHttp\Psr7\stream_for($resource);
  $client->request('GET', $url, ['save_to' => $stream]);
}

/**
 * Imagine this is using Laravel's Eloquent
 */
function recordVersion($version, $url, $desc) {
  \App\Downloads::create([
    'version' => $version,
    'url' => $url,
    'description' => $desc
    // created_at will be automatic for date reference
  ]);
}

/**
 * Just check if there are any others with this verison number.
 */
function isNewVersion($version) {
  return \App\Downloads::where(['version' => $version])->count() == 0;
}

```

## Status

"proposed"

## Consequences

### Pros

* This solution is completely automated
* It allows us to decide what format we use for internal tools, regardless of what the vendor uses.
* It provides possible oversight of package updates if pull requests are chosen

### Cons

* This requires maintenance of our own datastore for the update information
* It requires maintaining a new set of scripts and cron jobs, possibly a new database, etc. We'll have to monitor any tools used (like a framework/ORM for managing DB records) for security updates.
* It possibly relies on duplicated CI code in each application's repository (though this can be mitigated via the use of shared action configuration)
