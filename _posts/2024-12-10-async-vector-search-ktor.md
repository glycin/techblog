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

Pretty neat, right?! Without doing any labeling or pre-processing (except for loading the images into a vector database), we are able to show icons for any item the player might search for! The latency is also acceptable, but the user experience is horrible! They need to search for items one by one. What if we supported multiple input strings? To make it easy for the player (and most importantly, us) we will support a comma delimited input. So something like this: `potion, sword, table, chair, lamp`. Then we can just split the string and search for each item individually. We can even do this client side and fire up a request per item in the `search` list. **Ktor** is async by default so performance shouldn't be impacted at all in this use case. It looks like this:

{%
  include embed/video.html
  src='/assets/vid/multi_search.mp4'
  title='Multi input search video'
  autoplay=true
  loop=true
%}

Now as you can see this work quite well! Response times are under 100ms per request. Lets take a look using a comma-delimited list of 20 items one might find in an MMORPG.

```
Health Potion, Mana Potion, Elven Longbow, Dragonbone Sword, Mithril Armor, Magic Ring, 
Firestone, Healing Herb, Crystal Shard, Phoenix Feather, Iron Gauntlets, Teleportation Scroll, 
Enchanted Amulet, Steel Shield, Potion of Invisibility, Talisman of Luck, 
Rune Stone, Iron Sword, Cloak of Shadows, Greater Health Elixir, Ice Staff
```

If we do a call using this list as input, our server logging looks a little bit like this:

```
YYYY-MM-DD HH:MM:46.011 200 OK: GET - /icon/Health%20Potion in 72ms
YYYY-MM-DD HH:MM:46.051 200 OK: GET - /icon/Crystal%20Shard in 112ms
YYYY-MM-DD HH:MM:46.112 200 OK: GET - /icon/Healing%20Herb in 79ms
YYYY-MM-DD HH:MM:46.140 200 OK: GET - /icon/Talisman%20of%20Luck in 81ms
YYYY-MM-DD HH:MM:46.196 200 OK: GET - /icon/Phoenix%20Feather in 67ms
YYYY-MM-DD HH:MM:46.226 200 OK: GET - /icon/Mana%20Potion in 80ms
YYYY-MM-DD HH:MM:46.277 200 OK: GET - /icon/Iron%20Sword in 72ms
YYYY-MM-DD HH:MM:46.307 200 OK: GET - /icon/Iron%20Gauntlets in 72ms
YYYY-MM-DD HH:MM:46.350 200 OK: GET - /icon/Teleportation%20Scroll in 65ms
YYYY-MM-DD HH:MM:46.384 200 OK: GET - /icon/Enchanted%20Amulet in 69ms
YYYY-MM-DD HH:MM:46.421 200 OK: GET - /icon/Rune%20Stone in 65ms
YYYY-MM-DD HH:MM:46.458 200 OK: GET - /icon/Cloak%20of%20Shadows in 67ms
YYYY-MM-DD HH:MM:46.494 200 OK: GET - /icon/Potion%20of%20Invisibility in 65ms
YYYY-MM-DD HH:MM:46.547 200 OK: GET - /icon/Magic%20Ring in 71ms
YYYY-MM-DD HH:MM:46.585 200 OK: GET - /icon/Mithril%20Armor in 83ms
YYYY-MM-DD HH:MM:46.623 200 OK: GET - /icon/Elven%20Longbow in 71ms
YYYY-MM-DD HH:MM:46.660 200 OK: GET - /icon/Greater%20Health%20Elixir in 65ms
YYYY-MM-DD HH:MM:46.695 200 OK: GET - /icon/Dragonbone%20Sword in 64ms
YYYY-MM-DD HH:MM:46.739 200 OK: GET - /icon/and%20Ice%20Staff in 75ms
YYYY-MM-DD HH:MM:46.767 200 OK: GET - /icon/Firestone in 67ms
YYYY-MM-DD HH:MM:46.818 200 OK: GET - /icon/Steel%20Shield in 70ms
```

All our responses are within 100ms. Amazing! Unfortunately, there is an issue. Even though the server can handle our requests now, an implementation like this will eventually bring our server down (or force autoscaling to a point where we go bankrupt, I'm looking at you, AWS), once we inevitably have thousands of players doing thousands of searches!

This means we need to build an endpoint that receives a list of `searchTexts` and collects the results. Now, we’re smart enough to know that if we do this synchronously, it would take roughly 1.5 seconds to complete. Thankfully, we can use Kotlin Coroutines to speed things up a little bit. We just need to build a function that takes a list of `searchTexts` and maps them asynchronously to the results we find using Weaviate. Thankfully, using Kotlin extension functions, we can do that quite elegantly (shoutout to a good colleague of mine for this function).

```kotlin
suspend inline fun <T, R> Iterable<T>.asyncFlatMap(context: CoroutineContext = EmptyCoroutineContext, crossinline transform: suspend (T) -> List<R>): List<R> =
    withContext(context) {
        map { async { transform(it) } }.awaitAll().flatten()
    }
```
{: file='AsyncMap.kt'}

It might be a little bit intimidating at first, but this extension function basically creates a map of `Deferred<R>`, awaits them all, and returns the results of the transformations. We make sure it is an [inline function](https://kotlinlang.org/docs/inline-functions.html) as an optimization. Giving the option to pass a `CoroutineContext` will also give us the flexibility to switch to another coroutine context within this function if necessary. With this extension function at our disposal, we can collect our results like this:

```kotlin
suspend fun searchImagesAsync(texts: List<String>): List<Icon> = withContext (Dispatchers.IO) {
    texts.asyncFlatMap { text ->
        weaviateRepository.searchImageNearText(text, 10).map {
            it.toIcon()
        }
    }
}
```
{: file='IconService.kt'}

And when we search using the 20 items mentioned earlier, the result looks like this:

{%
  include embed/video.html
  src='/assets/vid/async_search.mp4'
  title='Async search video'
  autoplay=true
  loop=true
%}

Lets take a look at the response speed of the server. Of course isolated response speeds like this don't say much, but we can get a rough estimate if things are getting slower or faster after repeating the same call multiple times. 

```
YYYY-MM-DD HH:MM:12.102  200 OK: POST - /icon/with_list in 903ms
YYYY-MM-DD HH:MM:19.569  200 OK: POST - /icon/with_list in 904ms
YYYY-MM-DD HH:MM:20.822  200 OK: POST - /icon/with_list in 890ms
YYYY-MM-DD HH:MM:27.890  200 OK: POST - /icon/with_list in 894ms
YYYY-MM-DD HH:MM:32.726  200 OK: POST - /icon/with_list in 895ms
YYYY-MM-DD HH:MM:37.589  200 OK: POST - /icon/with_list in 897ms
YYYY-MM-DD HH:MM:41.630  200 OK: POST - /icon/with_list in 890ms
```
Hmm, the same call done 7 times always hovers around 900ms. So, getting all 200 icons we request asynchronously gives us an initial delay of 900ms, and then we have all the icons we asked for. It's not bad, but it's not great either. It’s definitely better than the 1.5 seconds or so that this would have taken if done synchronously, though! Using coroutines does add some overhead from potential coroutine switching, as well as from spinning up and managing the coroutines.

So, what can we do to improve this? Maybe there’s a way to stream results as we get them? Yes, that was a rhetorical question. There is a way, [Asynchronous Flow](https://kotlinlang.org/docs/flow.html).

### Streaming Vector Search

Async Kotlin Flows allow us to `emit` objects we compute in coroutines as they are ready. We can send these objects to a client one by one, eliminating any initial delay from the responses. Basically, it's a form of server side events (SSE), but preserving the ability to use good ol' `application/json` as our content type. Fortunately, thanks to **Ktor** and [`respondTextWriter`](https://api.ktor.io/ktor-server/ktor-server-core/io.ktor.server.response/respond-text-writer.html) we do not need to add and learn reactive frameworks like `Reactor` (or similar) and can use Flows natively. So how do we implement this?

```kotlin
call.respondTextWriter(ContentType.Application.Json) {
    iconService.searchImagesFlow(texts.searchTexts).collect { icon ->
        val item = Json.encodeToString(icon)
        LOG.info { "Sending image: ${item.takeLast(40)}" }
        write(item)
        write("\n")
        flush()
    }
}
```
{: file='Routing.kt'}

```kotlin
fun searchImagesFlow(texts: List<String>): Flow<Icon> = channelFlow {
    texts.forEach { text ->
        launch {
            weaviateRepository.searchImageNearText(text, 10).map {
                send(it.toIcon())
            }
        }
    }
}.flowOn(Dispatchers.IO)
```
{: file='IconService.kt'}

Less code than one would expect, right!? In our `Routing.kt`, we create an endpoint that responds using `respondTextWriter`, which collects the data in our `Flow` and writes it as it is received. We add a line break so that the client knows when a new object is sent and `flush()`, ensuring the items are sent immediately after they are written.

The function that creates our `Flow` is actually pretty similar to our `asyncFlatMap()` implementation. We loop through all the texts we receive, launch a new coroutine for each of them and `send` the icons as they are retrieved from our vector database. Furthermore we use a `channelFlow` builder instead of a `withContext` coroutine builder. Now, one with a keen eye might have noticed that we use the `channelFlow{}` builder instead of a `flow{}` builder. Why is that? Well, because the objects we want to `emit` (or `send`) are instantiated in a different coroutine than where the `Flow` is operating. Flows are not concurrent, but we can use the `channelFlow{}` builder to create a [`Channel`](https://kotlinlang.org/docs/channels.html) instead, which solves that issue for us. And thankfully, we just need to change `flow{}` to `channelFlow{}` and `emit()` to `send()` to make that work! Quite painless. So let's take a look at the result.

![Flow Stream](/assets/gif/stream.gif)

Exactly what we wanted! The millisecond we do a call, icons are beeing streamed to the client. Amazing...

On our game it looks like this.

{%
  include embed/video.html
  src='/assets/vid/flow_stream.mp4'
  title='Flow search video'
  autoplay=true
  loop=true
%}

And like this, we are ready to serve the thousands upon thousands of players that will flood our game! 

## Concluding...

So there we have it! We've seen how vector search and embedding models can spare us from tedious manual tagging and labeling work, and how **Ktor** and asynchronous Kotlin help us keep our services fast and responsive. Now unfortunately this isn't beeing used in a real MMORPG for that fast and smooth, immersive experience, but if you want to start building just that, you can find the code for the backend [here](https://github.com/glycin/ktor-vector-search).

So go on, let your imagination run wild, and craft the next big thing! With these tools and a bit of creativity, there's no limit to what we can achieve as software engineers. 

Good luck, and have fun!

## Additional viewing material
Still here? Then how about some more material for your viewing pleasure:
- [Introduction to vector databases](https://www.youtube.com/watch?v=5hAGhmHNTQQ)
- [Kotlin Coroutines explained](https://www.youtube.com/watch?v=gdUma9Q9Yz0)
- [Building apps with Ktor](https://www.youtube.com/watch?v=t7CoUvqzNrQ)
- [Multi modal search with Weaviate](https://www.youtube.com/watch?v=vgfIQEPYv_E)