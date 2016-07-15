#### Notes:

I've included two diagrams, a high-level block diagram and a simplified flow chart, to help facilitate the converstaion.

This is really, at its core, a very common solution for long-running tasks based on user requests and is applicable regardless of the end payload composition. You could be calculating arc-flash hazard boundaries or raster images of text and it will work out the same.

### Assumptions:

1. Some tasks may be long-running, such as in the case of image transformations
2. Some assets, such as previously transformed images, may be reused
3. There are no requirement that this run as a monolithic process; a microservice/SOA is acceptable

### Architecture

<img src=https://github.com/jason-c-child/taskapalooza/blob/master/Screen%20Shot%202016-07-15%20at%2011.38.39%20AM.png width=50% height=50%>

The system is comprised of the following components:

1. HTTPS Ingress
2. Cache Server
3. Task Queue
4. Database
5. Out-of-band services (text and image processing)

#### HTTPS Ingress

This represents all the front-facing input, such as reverse proxy and load balancing
in addition to the process that fetches the requested resources or schedules their
out-of-band processing.

#### Cache Server

The cache server allows quick response for previously generated resources,
such as processed images.

#### Task Queue

Using a task queue for long-running, out-of-band jobs such as image
processing will allow you to distribute load as well as return placeholder data
or job status

#### Database

Given _Assumption 2_, that assets may be reused, it would be wise to persist this
in a database. The duration of persistence is debatable.

#### Out-of-band Services

Raster image generation from text assets, and image transformations such as
downsampling.

### Flow

<img src=https://github.com/jason-c-child/taskapalooza/blob/master/Screen%20Shot%202016-07-15%20at%2011.39.23%20AM.png width=50% height=50%>

#### Pre-cached

1. Accept payload
2. Parse payload
3. Check cache
4. Return cached assets

#### Stored

1. Accept payload
2. Parse payload
3. Check cache
4. Check database
5. Return stored assets
6. Load assets in cache

#### New Assets

1. Accept payload
2. Parse payload
3. Check cache
4. Check database
5. Schedule task to process assets

### Considerations Addressed

#### How is work divided among the components?
Either the assets are cached/stored or they are not. If they are not then a task
is queued to perform the processing so the results can be stored and cached for later
use.

#### How many distinct services or libraries does this require to be built?
Four, since you could use the same service as the cache and the task queue.
If you were adventurous, or wanted to reduce the number of services, you could
use a single service (mongodb, for instance) to house the job queue backend and
drop the caching layer.

#### How will this scale?
Horizontally. The bottlenecks are at the ingress point and the out-of-band processing (though
that is less critical than the ingress since tasks get queued).

#### What ways can this system fail and how would we handle that?
This is a rather loaded question; technically it can fail anywhere. By using a distributed
task queue we can build redundancy and fault tolerance into the system. This also lets you
guard against conditions such as sudden spikes in demand (I'm told this is the Thundering Herd problem).

#### What security concerns do we have?
Since the majority of the action happens behind the HTTPS ingress point there is not much that
I can see here, though I am no web security expert. I do feel that this architecture allows for
very small API surface...which I would assume translates into a smaller attack surface.

#### How could we test this system?
Each component can be tested in isolation with various degrees of mocking. The out-of-band
services obviously enjoy the most autonomy, however each component is highly decoupled and
depends on an eventual universal source of truth; the database.

#### Is there a need for storing state? What would that look like?
Yes, because the long running process issue it is handy to store state and by using the task
queue we enjoy that ability baked in. This allows a request for assets to provide
status updated to the rest of the system.

#### Are there specific technologies that would fit here better than others?

I'd recommend using Redis for the caching server. Celery (python) or [Kue](https://github.com/Automattic/kue) (node) for task management. For a database
it really doesn't matter much in my thinking; something with low latency reads would be ideal. Given
that all our long running jobs are out-of-band then it seems like Node/Express/Koa would work well for
handling the requests. That said, AWS has a swarm of managed services that could be exploited to
stitch this together; SQS and Elastic Beanstalk (and to a lesser degree Lambda) come to mind. After all,
not everyone wants to feed and water a cluster of Redis servers...
