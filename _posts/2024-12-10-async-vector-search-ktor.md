---
layout: post
title: "Fast async vector search with Ktor and Weaviate"
date: 2024-12-10
categories: [technical, coding, ktor]
---

![Banner](/assets/gif/banner.gif)

## Imagine...

Imagine that we are developing an MMORPG (massively multiplayer online role-playing game). Our differentiating feature is that users can craft and store pretty much anything they can come up with! Quite ambitious, but that is how we roll in the software engineering world. Ambition and scope creep are our middle names! One of the first issues we face is the fact that whenever a user crafts, buys, or finds something, we want to display an icon for that item. We are developing a game, after all... we can't just show text to our players!

There is no way we are going to create a custom icon for all possible items in our game. Tagging icons is also out of the question, because that sounds like manual work, and we developers don't like doing things manually! So what’s the solution? Well, thanks to AI and machine learning, that would be **embedding models** and **vector search**!

## Vector search
Vector search is the art of using embedded vectors to search for semantically similar items. It is how LLMs like ChatGPT work (among other techniques) and has become quite popular in recent years. Embedding models transform objects into their vector representations, which are arrays of *float* numbers. These models have been trained using machine learning to understand the context and meaning of whatever they are embedding, and they have become quite good at what they do. So what does that do for us? Well, we can use an embedding model that understands images and text (a so-called multi-modal embedding model) and use that to find an icon that is semantically close to whatever the player is searching for! Then all we need to do is a vector search using the input text of the player and return the icon that best fits that text.

To do this, we need a backend that receives a `searchText` and a vector database that we can leverage for quick and painless vector searches. In our case, we will use **Kotlin** and **Ktor** for our backend and **Weaviate** as a vector database.

## Setting up Weaviate
There are a lot of vector databases out there, so why are we choosing [Weaviate](https://weaviate.io/)? Well, most importantly, ease of use. Weaviate is open-source and allows us to easily deploy the database (locally) using *Docker*. They even have a handy *docker-compose* [generator](https://weaviate.io/developers/weaviate/installation/docker-compose)! Clicking through it gave us the the following `docker-compose.yml`:

```yaml
---
services:
  weaviate:
    command:
    - --host
    - 0.0.0.0
    - --port
    - '8080'
    - --scheme
    - http
    image: cr.weaviate.io/semitechnologies/weaviate:1.27.0
    ports:
    - 8080:8080
    - 50051:50051
    volumes:
    - C:\Projects\ktor-vector-search\weaviate_volume:/var/lib/weaviate
    restart: on-failure:0
    environment:
      CLIP_INFERENCE_API: 'http://multi2vec-clip:8080'
      QUERY_DEFAULTS_LIMIT: 25
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'true'
      PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
      DEFAULT_VECTORIZER_MODULE: 'multi2vec-clip'
      ENABLE_MODULES: 'multi2vec-clip'
      CLUSTER_HOSTNAME: 'node1'
  multi2vec-clip:
    image: cr.weaviate.io/semitechnologies/multi2vec-clip:sentence-transformers-clip-ViT-B-32
    environment:
      ENABLE_CUDA: '0'
...
```

Quite neat right?! We can just run `docker compose up`, and we’ll have an instance running locally in no time! There are some interesting lines here:

```yaml
    volumes:
    - C:\Projects\ktor-vector-search\weaviate_volume:/var/lib/weaviate
```

Here we define a persistent volume for our vector database. That means there won’t be any data loss if we happen to delete the Docker image, as some of us might have experienced before...

```yaml
  multi2vec-clip:
    image: cr.weaviate.io/semitechnologies/multi2vec-clip:sentence-transformers-clip-ViT-B-32
```
This is an important line. Here we tell Weaviate that we want to use a `clip` model (with a link to where to download the Docker image from). Clip models are multi-modal, which is exactly what we need! Now, every time we push something to Weaviate, it will automatically embed that data using this exact model. That means we don’t have to do any embedding ourselves. Everything will be taken care of by our vector database.

Now we just need to create our vector space (or schema) for the objects that we will be storing. This is like defining a table in a relational database. We can do this by making a `POST` request to the Weaviate API:

```
curl --request POST \
  --url http://localhost:8080/v1/schema \
  --header 'content-type: application/json' \
  --data '{
  "class": "Icon",
  "description": "A schema to store icons",
  "vectorizer": "multi2vec-clip",
  "moduleConfig": {
    "multi2vec-clip": {
      "textFields": [
        "text"
      ],
      "imageFields": [
        "image"
      ]
    }
  },
  "properties": [
    {
      "dataType": [
        "text"
      ],
      "name": "text",
      "multi2vec-clip": {
        "skip": false,
        "vectorizePropertyName": false
      }
    },
    {
      "dataType": [
        "blob"
      ],
      "name": "image",
      "multi2vec-clip": {
        "skip": false,
        "vectorizePropertyName": false
      }
    }
  ]
}'
```
Here we create a schema with the `className` "Icon" (class names always have to start with an uppercase letter) and let Weaviate know to use the `clip` model to vectorize the fields. We also define two fields: `text` and `image`, which, unsurprisingly, will hold textual and image data. `Text` will be simple strings, while `image` will be base64-encoded images.

And that's it! We are ready to ingest and query data from our Weaviate instance. Now let's make some endpoints to do just that!

## Ktor
Since we are developing a game here (and not just any game, an MMORPG), we need our search to be *fast* as lightning and as *async* as we can get it! Players might be searching for a lot of items at any given time, so async will help us get those icons to the player as quickly as possible! That means [Ktor](https://ktor.io/) is probably our best choice. Ktor heavily leverages *Kotlin coroutines*, so calls are async by default. That’s great! Exactly what we need! And the compact way of declaring our routing is just a nice plus.

### Setting up the service
Setting up our service is quite straightforward. Our configuration isn’t that exciting. We set up our `ContentNegotiation` for JSON, configure some log levels, and create our services to inject them into our routes. We do this manually since using a DI framework just for a few classes is really kinda overkill. Don’t forget, people: less is more.

```kotlin
fun Application.module() {

    install(ContentNegotiation) {
        json(Json {
            prettyPrint = true
            isLenient = false
            ignoreUnknownKeys = true
        })
    }

    install(CallLogging) {
        level = Level.WARN
    }

    val repository = WeaviateRepository()
    val iconService = IconService(repository)

    configureRouting(iconService)
}
```
{: file='Application.kt'}

The Weaviate repository we create here is using the [**Weaviate** Java client](https://weaviate.io/developers/weaviate/client-libraries/java). This allows us to relatively easily interface with the vector database running on our local Docker environment. If we didn’t want to use the Weaviate Java client, we would either need to make REST calls ourselves or use an abstraction framework like Langchain4j.

### Loading icons in the vector database
![Icons](/assets/gif/icons.gif)

But which icons are we going to ingest in our database? Well, thanks to some [humble bundles](https://www.humblebundle.com/), there is a healthy amount of icons that we can use—1029 icons to be precise, stored in a local directory. These specific icons are unfortunately subject to copyright laws and can’t be shared online, but any [icon collection](https://opengameart.org/content/496-pixel-art-icons-for-medievalfantasy-rpg) can do the trick! Now, all we need to do is create an endpoint that gets a directory path (or URL) and loads all these icons into our vector database:

```kotlin
post("/ingest/directory") {
    try {
        val path = call.receive<IconDirectoryPath>()
        call.respond(iconService.loadImagesInDb(path.path))
    } catch (e: JsonConvertException) {
        LOG.info { "Couldn't parse IconDirectoryPath: ${e.message}" }
        call.respond(HttpStatusCode.BadRequest)
    }
}
```
{: file='Routing.kt'}

```kotlin
fun loadImagesInDb(imageDirectory: String): Boolean {
    LOG.info { "Loading in images from $imageDirectory" }
    val directory = File(imageDirectory)
    val uris = directory
        .walk()
        .filter { it.isFile && it.extension == "png" }
        .map { it.toURI() }
        .toList()
    LOG.info { "Finished in images from $imageDirectory" }
    return weaviateRepository.batchAddImages(uris)
}
```
{: file='IconService.kt'}

```curl
curl --request POST \
  --url http://127.0.0.1:1337/icon/ingest/directory \
  --header 'content-type: application/json' \
  --data '{
  "path": "C:\\Projects\\ktor-vector-search\\vectorData"
}'
```
The above snippets should be quite straightforward (otherwise, I have failed you...). We `POST` a directory path, get the `URI` of each image in that directory, and batch add them to Weaviate. Once we run this, all images in the specified directory will be loaded into Weaviate! In this case, that will take roughly a minute and a half. Thankfully, we only need to do this once, thanks to the persistent volume we set up earlier. And that’s it! Now we can finally start searching for icons using text!

### Searching for images using text
After some setup, we’re all ready to build our search endpoints.

```kotlin
route("/icon") {
    get("/{searchText}") {
        val searchText = call.parameters["searchText"]
        if (searchText.isNullOrBlank()) {
            call.respond(HttpStatusCode.BadRequest)
            return@get
        }

        call.respond(iconService.searchImage(searchText))
    }
}
```
{: file='Routing.kt'}

A simple `GET` will do the trick here. We can extract our search text from the path and send it to our `IconService.kt`. The `IconService` will then forward the request to our `WeaviateRepository`. 

```kotlin
fun searchImage(text: String): List<Icon> {
    return weaviateRepository.searchImageNearText(text, 10).map {
        it.toIcon()
    }
}
```
{: file='IconService.kt'}

In the `WeaviateRepository` is where most the magic happens. Here we tell Weaviate how to perform the vector search exactly.

```kotlin
val result = client.graphQL()
            .get()
            .withClassName(WeaviateClassNames.ICON_CLASS)
            .withNearText(
                NearTextArgument.builder()
                    .concepts(arrayOf(text))
                    .distance(0.9f)
                    .build()
            )
            .withLimit(limit)
            .withFields(Field.builder().name("image").build())
            .run()
```
{: file='WeaviateRepository.kt'}

This might look alien, but once "decoded" it makes sense. Since Weaviate uses `GraphQL` for their search API, we use a builder to create a `GraphQL` request. We want to do a `get()` request on the vector schema with the class name `Icon`. There, we want to search for any vectors that are close to the concept of `text`, where `text` is the `string` that we supply. We limit our results to vectors that are within a distance of `0.9`. Identical vectors will have a distance of `0`, and opposite vectors will have a distance of `2`. We tell Weaviate to limit the results to 10 and to only return the image field of the found objects. See, makes perfect sense!

And with this, we are ready to search! Players can type anything they want, and we’ll most likely have an icon to show them! It looks something like this:

{%
  include embed/video.html
  src='/assets/vid/simple_search.mp4'
  title='Search video'
  autoplay=true
  loop=true
%}

Pretty neat, right?! Without doing any labeling or pre-processing (except for loading the images into a vector database), we are able to show icons for any item the player might search for! The latency is also acceptable, but the user experience is horrible! They need to search for items one by one. What if we supported multiple input strings? To make it easy for the player (and most importantly, us) we will support a comma delimited input. So something like this: `potion, sword, table, chair, lamp`. Then we can just split the string and search for each item individually. **Ktor** is async by default so performance shouldn't be impacted. It looks like this:

{%
  include embed/video.html
  src='/assets/vid/multi_search.mp4'
  title='Search video'
  autoplay=true
  loop=true
%}

## Additional viewing material