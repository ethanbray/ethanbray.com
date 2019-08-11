---
layout: post
title:  "Streaming Buffered PHP Output in Zend Framework"
date:   2019-08-11 12:00:00 +0000
categories: ["zend"]
---

I was recently working on a feature that required me to retrieve large amounts 
of data (7 figure document counts) from Elasticsearch, modify the 
data and then output a CSV file. This feature needed to be performant and not have 
an adverse effect on the memory usage of the application's EC2 instance.

An original implementation of the feature retrieved all documents, stored them
in memory and then created a file on disk that would be served to the user.
This resulted in memory exhaustion, as well as taking up large amounts of disk
space on production instances.

This pseudocode represents the initial implementation we had:

```php
class ExportController() {
    public function exportAction(): Stream {
        $exportQuery = $this->params()->fromPost('query');
        $this->dataExporter->export($exportQuery);
        
        $response        = $this->getResponse();
        $responseHeaders = $response
            ->getHeaders()
            ->addHeaders([
                'Content-Description'       => 'File Transfer',
                'Content-Type'              => 'text/csv',
                'Content-Transfer-Encoding' => 'binary',
                'Expires'                   => 0,
                'Cache-Control'             => 'must-revalidate',
                'Pragma'                    => 'public',
            ]);
        $response->setHeaders($responseHeaders);
        $response->setContent($fileContents);
        return $response;
    }
}

class DataExporter {
    public function export($exportQuery)
    {
        // Get all of the results
        $results = $this->elasticsearchService->getData($exportQuery);
        
        // Iterate through data and output to CSV file
        $filePointer = fopen('/tmp/data.csv', 'wb');
        foreach ($results as $result) {
            fputcsv($filePointer, $result, ',');
        }
    }
}
```

[`fputcsv`](https://www.php.net/manual/en/function.fputcsv.php) takes a `resource` 
as its first parameter, including a resource that points to a PHP I/O stream such 
as `php://output`. By replacing `/tmp/data.csv` with `php://output` we removed our
need to create CSV files on the server.

Now that we're writing the data straight to the output stream, we need to send the 
correct headers before we flush the buffer:

```php
class ExportController() {
    public function exportAction(): Stream {
        $exportQuery = $this->params()->fromPost('query');
        $this->dataExporter->export($exportQuery);
    }
}

class DataExporter {
    public function export($exportQuery): void
    {
        // Get all of the results
        $results = $this->elasticsearchService->getData($exportQuery);
        
        // Iterate through data and output to CSV file
        ob_start();
        $outputPointer = fopen('php://output', 'wb');
        foreach ($results as $result) {
            fputcsv($outputPointer, $result, ',');
        }
        
        // Close our file pointer, send the file headers and flush our output buffer
        fclose($filePointer);
        header('Content-Description: File Transfer');
        header('Content-Type: text/csv');
        header('Content-Transfer-Encoding: binary');
        header('Expires: 0');
        header('Cache-Control: must-revalidate');
        header('Pragma: public');
        ob_end_flush();
        exit();
    }
}
```

We use `exit()` to avoid a `headers already sent` error as Zend Framework will try 
to send a response, and more headers, to the user.

If we were to write a test for the `export` method, we'd swiftly run into  the issue 
where our usage of `exit()` ends our PHPUnit process. Short of monkey patching the 
`exit()` function, there's not much we can do about this. Other than removing `exit()`
that is. Instead of flushing the buffer in our `export` method, we can instead create
a `Stream` response and return that to our controller. The controller will then return
the `Stream` to Zend Framework which will send it to our user.

```php
class ExportController() {
    public function exportAction(): Stream {
        $exportQuery = $this->params()->fromPost('query');
        return $this->dataExporter->export($exportQuery);
    }
}

class DataExporter {
    public function export($exportQuery): Stream
    {
        // Get all of the results
        $results = $this->elasticsearchService->getData($exportQuery);
        
        // Create our Stream
        $outputPointer = fopen('php://output', 'wb');
        $stream = new Stream();
        $stream->setStream($outputPointer);
        $stream->setHeaders((new Headers())->addHeaders([
            'Content-Description'       => 'File Transfer',
            'Content-Type'              => 'text/csv',
            'Content-Transfer-Encoding' => 'binary',
            'Expires'                   => '0',
            'Cache-Control'             => 'must-revalidate',
            'Pragma'                    => 'public',
        ]);
        
        // Iterate through data and output to CSV file
        foreach ($results as $result) {
            fputcsv($outputPointer, $result, ',');
        }
        
        return $stream;
    }
}
```

When a response is returned by a controller, Zend MVC emits a `SendResponseEvent`
event that is listened to by several Zend Framework listeners. A listener is available
for each type of response we may return. In our case, the `StreamResponseSender`
will be responsible for sending our response onwards.

- `Zend\Mvc\SendResponseListener\PhpEnvironmentResponseSender` with a priority of `-1000`
- `Zend\Mvc\SendResponseListener\ConsoleResponseSender` with a priority of `-2000`
- `Zend\Mvc\SendResponseListener\SimpleStreamResponseSender` with a priority of `-3000`

```php
class SimpleStreamResponseSender extends AbstractResponseSender
{
    public function sendStream(SendResponseEvent $event)
    {
        if ($event->contentSent()) {
            return $this;
        }
        $response = $event->getResponse();
        $stream   = $response->getStream();
        fpassthru($stream);
        $event->setContentSent();
    }

    public function __invoke(SendResponseEvent $event)
    {
        $response = $event->getResponse();
        if (! $response instanceof Stream) {
            return $this;
        }
        $this->sendHeaders($event);
        $this->sendStream($event);
        $event->stopPropagation(true);
        return $this;
    }
}
```

Now unfortunately, `fpassthru` does not support resources using `php://output`.
Attempting to use the `StreamResponseSender` will result in an error about an 
invalid resource. Replacing `fpassthru` with `ob_end_flush` is all that is required
to make the `SimpleStreamResponseSender` work.

Now we'll need to create our own response sender to handle these `Stream` responses
using `php://output`. We also need to differentiate our responses using `php://output`
from a standard `Stream` response.

```php
class BufferedStream extends Stream {
    public function __construct($filePointer)
    {
        $this->setHeaders((new Headers())->addHeaders([
            'Content-Description'       => 'File Transfer',
            'Content-Type'              => 'text/csv',
            'Content-Transfer-Encoding' => 'binary',
            'Expires'                   => '0',
            'Cache-Control'             => 'must-revalidate',
            'Pragma'                    => 'public',
        ]));
    }
}

class BufferedStreamResponseSender extends AbstractResponseSender
{
    public function sendStream(SendResponseEvent $event): self
    {
        if ($event->contentSent()) {
            return $this;
        }
        
        // fpassthru does not work with php://output so we have to flush the buffer instead
        ob_end_flush();
        $event->setContentSent();
        return $this;
    }

    public function __invoke(SendResponseEvent $event)
    {
        $response = $event->getResponse();
        if (! $response instanceof BufferedStream) {
            return $this;
        }
        
        $this->sendHeaders($event);
        $this->sendStream($event);
        $event->stopPropagation(true);
        return $this;
    }
}
```

Now we have our listener and our new `BufferedStream`, we simply need to update our
`DataExporter` to use the correct response and attach our listener to the `SendResponseEvent`.

```php
class ExportController() {
    public function exportAction(): BufferedStream {
        $exportQuery = $this->params()->fromPost('query');
        return $this->dataExporter->export($exportQuery);
    }
}

class DataExporter {
    public function export($exportQuery): BufferedStream
    {
        // Get all of the results
        $results = $this->elasticsearchService->getData($exportQuery);
        
        // Create our Stream
        $outputPointer = fopen('php://output', 'wb');
        $stream = new BufferedStream();
        $stream->setStream($outputPointer);
        
        // Iterate through data and output to CSV file
        foreach ($results as $result) {
            fputcsv($outputPointer, $result, ',');
        }
        
        return $stream;
    }
}

class Module implements ConfigProviderInterface
{
    /**
     * @inheritdoc
     */
    public function getConfig()
    {
        return include __DIR__ . '/../config/module.config.php';
    }

    /**
     * @inheritdoc
     */
    public function onBootstrap(EventInterface $e)
    {
        /** @var EventManager $eventManager */
        $eventManager  = $e->getApplication()->getEventManager();
        $eventManager->attach(SendResponseEvent::EVENT_SEND_RESPONSE, new BufferedStreamResponseSender(), 1);
    }
}
```

We've now removed any need for processing data in to an intermediary
file. Data is processed and then streamed straight to PHP's buffered
output.

Elasticsearch offers the [Scroll API](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/search-request-body.html#request-body-search-scroll)
for processing large amounts of data. It operates in a similar way to a 
cursor on a traditional database. By replacing our single `getData`
call with a loop utilising the Scroll API, we can minimise the amount
of data we have stored in memory at one time.

```php
class DataExporter {
    public function export($exportQuery): BufferedStream
    {
        // Create our Stream
        $outputPointer = fopen('php://output', 'wb');
        $stream = new BufferedStream();
        $stream->setStream($outputPointer);

        // Start our scroll
        $resultSet = $this->elasticsearchService->startScroll($exportQuery);
        $this->writeDataToResource($outputPointer, $resultSet);

        // Continue the scroll till we have no further results
        while (count($resultSet->getResults()) !== 0) {
            $resultSet = $this->elasticsearchService->continueScroll($resultSet);
            $this->writeDataToResource($outputPointer, $resultSet);
        }
        
        return $stream;
    }
    
    private function writeDataToResource($outputPointer, ResultSet $resultSet): void
    {
        $results = $resultSet->getResults();
        foreach ($results as $result) {
            fputcsv($outputPointer, $result, ',');
        }
    }
}
```

By utilising streams and the Scroll API, we reduced our memory usage and removed
our file usage completely.