# Apache Beam Workshop
Apache Beam workshop for the Streaming Specialty Course.

This workshop's intention is to build a streaming solution using GCP's managed tools: [Pub/Sub](https://cloud.google.com/pubsub) (message queue), [Dataflow](https://cloud.google.com/dataflow) (A managed runner for Apache Beam), [Bigtable](https://cloud.google.com/bigtable) (datalake or staging database) and [BigQuery](https://cloud.google.com/bigquery) (Data Warehouse).

The workshop will be divided into two parts: the first one builds a pipeline that reads data from `Pub/Sub` and inserts it into `BigQuery`, and the second part builds a pipeline that ingests data to `Bigtable` and another pipeline to read data from `Bigtable` and inserts it into `BigQuery`.

> ARCHITECTURE PICTURE
> Lecturas de columnar databases

## Setup
For this workshop, we'll need:

- Access to GCP's resources. For that, GCP's free tier will suffice.
- [Python 3](https://www.python.org/downloads/).
- An editor. [PyCharm](https://www.jetbrains.com/pycharm/download/) is a good option.
- [Git](https://git-scm.com/) if you wish to clone this repository instead of following along.

With this, we're all set to start the workshop.

## Part 1

This part builds a pipeline to read data from `Pub/Sub` and write to `BigQuery`.

### Step 1: More setup
Create a folder for your project. In that folder, create a file named `requirements.txt` with a single line:
```
apache-beam[gcp]
```

If you open your folder on `PyCharm` and open the file in your editor, it will prompt you to install the requirements. Otherwise, you can run `pip install -r requirements.txt` on your terminal.

Next up, we need to create a `Pub/Sub` topic and subscription. If you navigate to [Pub/Sub](https://console.cloud.google.com/cloudpubsub/), you can find the `topics` list on the left panel. Create a new topic and it will prompt you to create a subscription for the topic. You can use all the default settings for both. Note the subscription's name.

### Step 2: First pipeline
In your project folder, create a Python package using `PyCharm`, or just create an empty folder and create an empty file inside it called `__init__.py`. Create another file for our pipeline with any name you'd like, maybe `dataflow.py`.

Let's start building our pipeline. We need a `run` function like this:
```python
import apache_beam as beam

def run(argv=None, save_main_session=True):
	"""
	Builds and runs pipeline
	"""
	
if __name__ == '__main__':
	run()
```

Next up, we create our actual pipeline. That's an object that receives a series of options that can range from resources settings to configurations or custom parameters. In our case, we want our pipeline to be a streaming pipeline, so we'll add these imports:

```python
from apache_beam.options.pipeline_options import PipelineOptions
from apache_beam.options.pipeline_options import StandardOptions
```

And we'll add the following lines inside our `run` function:
```python
    pipeline_options = PipelineOptions(pipeline_args)
    
    std_options = pipeline_options.view_as(StandardOptions)
    std_options.runner = 'DirectRunner'
    std_options.streaming = True
    
    pipeline_options.view_as(StandardOptions).streaming = True
    pipeline = beam.Pipeline(options=pipeline_options)
```
Note that `DirectRunner` uses your local machine to run the pipeline. This is useful for quick testing.

Now, remember I told you to note your `Pub/Sub` subscription name? We'll need that next, so save it to a variable. It needs to come in the following format: `pubsub = 'projects/<PROJECT_ID>/subscriptions/<SUBSCRIPTION_NAME>'`. Note that `PROJECT_ID` is not necessarily your project's name; if you can't locate it, refer to [this page](https://support.google.com/googleapi/answer/7014113?hl=en).

Add these lines to our `run` function:
```python
    (pipeline
     | 'read' >> beam.io.ReadFromPubSub(subscription=pubsub).with_output_types(bytes)
     | 'decode' >> beam.Map(lambda x: x.decode('utf-8')))
     
   result = pipeline.run()
```

This is Beam's syntax for applying transformations to a pipeline. This would be equivalent to something convoluted along the lines of:
```python
    pipeline.apply(beam.io.ReadFromPubSub(subscription=pubsub).with_output_types(bytes), 'read').apply(beam.Map(lambda x: x.decode('utf-8')), 'decode')
```

Each step in the pipeline returns a `PCollection` object, which we can think of as a list of a single type. In this case, our `read` step returns a `PCollection` of bytes, and our `decode` step returns a `PCollection` of strings.

To test the pipeline so far, you can publish a message to `Pub/Sub` manually using the "Publish Message" option on the top part of the topic's page. To check that the message was read, you can wait about a minute for the subscription's graph to update:

![image](https://user-images.githubusercontent.com/92763131/164755931-051d0bcb-09c6-4392-b6d9-f9c82b88db41.png)

### Step 3: Understanding Apache Beam

The pipeline we built won't print anything or notify us about messages being read. We could try to change our pipeline to include a print like this:
```python
    p_collection = (pipeline
                    | 'read' >> beam.io.ReadFromPubSub(subscription=pubsub).with_output_types(bytes)
                    | 'decode' >> beam.Map(lambda x: x.decode('utf-8')))
     
    print(p_collection)
```

This won't print messages. To print messages, we need a `PTransform` that prints each message as it finds it in the collection:
```python
    p_collection = (pipeline
                    | 'read' >> beam.io.ReadFromPubSub(subscription=pubsub).with_output_types(bytes)
                    | 'decode' >> beam.Map(lambda x: x.decode('utf-8')))
                    | 'print' >> beam.Map(lambda x: print(x)))
    print(p_collection)
```

### Step 4: First DoFn

A `PTransform` can be any of [these](https://beam.apache.org/documentation/transforms/python/elementwise/pardo/) on the left panel. One such transform is the `ParDo` transform, that works alongside a `DoFn` function. It runs the function for each element in the `PCollection`. A great advantage of these `DoFns` is that they can be user-defined functions. Let's define our own function.

This can be done in another file called `print_dofn.py` for example (recommended), or this can be done in the same file.

```python
class PrintDoFn(beam.DoFn):
    """
    Prints each element in the PCollection
    """

    def process(self, element):
        """
        This will run for each element in the collection
        """
        print(element)
```

With this new `DoFn`, we can change our 'print' transform like this:
```python
...
| 'print' >> beam.ParDo(PrintDoFn()))
```
