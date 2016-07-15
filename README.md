### Assumptions:

1. Some tasks may be long-running, such as in the case of image transformations
2. Some assets, such as previously transformed images, may be reused
3. There are no requirement that this run as a monolithic process; a microservice/SOA is acceptable

### Architecture

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



