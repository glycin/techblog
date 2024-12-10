---
layout: post
title: "Fast async vector search with Ktor and Weaviate"
date: 2024-12-10
categories: [technical, coding, ktor]
---

![Banner](/assets/gif/banner.gif)

## Imagine...

Imagine that we are developing a MMORPG (massively multiplayer online role-playing game). Our differentiating feature is that users can craft and store pretty much anything they can come up with! Quite ambitious, but that is how we roll in the software engineering world. Ambition and scope creep is our middle name! 
One of the first issues we face is the fact that whenever a user crafts, buys or finds something, we want to display an icon for that item. We are developing a game in the end... can't just show text to our players! 

There is no way we are going to create a custom icon for all possible items in your game. Tagging icons is also out of the questions, because that sounds like manual work, and we developers don't like doing things manually! So what is the solution? Well, thanks to AI and machine learning, that would be **embedding models** and **vector search**! 

## Vector search
Vector search is the art of using embedded vectors to search for semantically similar items. It is how LLM's like chatGPT work (among other techniques) and has become quite popular past years. Embedding models transform objects into their vector representation, which is an array of *float* numbers. These models have been trained using machine learning to understand the context and meaning of whatever they are embedding, and have become quite good at what they do. So what does that do for us? Well we can use an embedding model that understands images and text (a so called multi-modal embedding model) and use that to find an icon that is semantically close to whatever the player is searching! Then all we need to do is a vector search using the input text of the player, and return the icon that best fits that text. 

To do this we just need a backend that receives a `searchText` and a vector database that we can leverage for quick and painless vector search. In our case we will use **Kotlin** and **Ktor** for our backend and **Weaviate** as a vector database.

## Setting up Weaviate
There are alot of vector databases out there, so why are we choosing [Weaviate](https://weaviate.io/)? Well, most importantly, ease of use. Weaviate is open source, and allows us to easily deploy the database (locally) using *docker*. They even have a handy *docker-compose* [generator](https://weaviate.io/developers/weaviate/installation/docker-compose)! Clicking through it gave us the the following `docker-compose.yml`:

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

Quite neat right?! We can just run `docker compose up` and we will have an instance running locally in no time! There are some interesting lines here:

```yaml
    volumes:
    - C:\Projects\ktor-vector-search\weaviate_volume:/var/lib/weaviate
```

Here we define a persistent volume for our vector database. That means there wont be any data loss if we happen to delete the image, which has never happened before...

```yaml
  multi2vec-clip:
    image: cr.weaviate.io/semitechnologies/multi2vec-clip:sentence-transformers-clip-ViT-B-32
```

This is an important line. Here we tell Weaviate that we want to use a `clip` (with a link to where to download the image from) model. Clip models are multi-modal, exactly what we need! Now everytime we push something to Weaviate, Weaviate will automatically embed that data using this exact model. That means we dont have to do any embedding ourselves, everything will be taken care of by our vector database.

Now we just need to create our vector space (or schema) for the objects that we will be storing. This is like defining a table in a relational database. We can do this by doing a `POST` request to the Weaviate API:

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

Here we create a schema with `className` `Icon` (class names always have to start with an uppercase letter), and let Weaviate know to use the `clip` model to vectorize the fields. We also define two fields, `text` and `image`, which unsuprisingly will hold textual and image data. `Text` will be simple strings, while `image` will be base64 encoded images.

And that's it! We are ready to ingest and query data from our Weaviate instance. Now let's make some endpoints to do just that!

## Ktor
Since we are developing a game here (and not just any game, a MMORPG), we need our search to be *fast* as lightning and as *async* as we can get it! Players might be searching for alot of items at any given time, so *async* will help us get those icons to the client as quickly as possible! That means [Ktor](https://ktor.io/) is probably our best choice. Ktor is heavily leveraging *kotlin coroutines* to be async by default. That's great! Exactly what we need! And the compact way of declaring our routing is just a nice plus.




## Additional viewing material