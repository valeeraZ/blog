---
title: 'Talk with your code in GitHub using llama-index'
date: '2024-07-27'
lastmod: '2024-04-27'
tags: ['Python']
draft: false
images: ['/static/images/llama_github.png']
layout: PostLayout
summary: 'Use embedding and vector search to ask questions and get answers from your code in GitHub'
---

Inspired from many posts talking about [RAG](https://docs.llamaindex.ai/en/stable/use_cases/q_and_a/) with code:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Let&#39;s build a &quot;Chat with your code&quot; RAG application, step-by-step:</p>&mdash; Akshay 🚀 (@akshay_pachaar) <a href="https://twitter.com/akshay_pachaar/status/1765716620847300689?ref_src=twsrc%5Etfw">March 7, 2024</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

And this post [github リポジトリを Embedding して質問に答えてもらう](https://note.com/nike_cha_n/n/n1d2f5ee81644)

I am also very interested in this topic and want to improve the solution with 2 main points:

- Imagine to make embedding for a large file of code, how can we split the code into reasonal smaller parts, like divide a Python file by divide classes/function and make the embedding for each part? We can use the AST tree to split the code into smaller chunks and make the embedding for each part.
- For a repository containing multiple files with different languages, we need to use different AST tree splitter intelligently.

I will use `llama-index` to implement this idea in a Jupiter notebook in this post.

By default in the following code, `llama-index` will use OpenAI API, with **text-embedding-ada-002** model to generate the embeddings for the code files, and with OpenAI **gpt-3.5-turbo** model to generate the response for the questions.

I will try to create embeddings for my repository [semantic-search](https://github.com/valeeraZ/semantic-search) who uses FastAPI to create a semantic search engine. I will then ask technical questions about the code in the repository and try to get the answers.

## Prepare the environment

1. To read the code from your repository, you will need to have a general token with `repo` scope. Go to your [GitHub settings](https://github.com/settings/tokens/new) to create a new token with `repo` scope then you can save the token as a environment variable `GITHUB_TOKEN`.

2. you would also need an `OPENAI_API_KEY` to use the OpenAI API.

3. launch a local PostgreSQL database to store the embedding of the code. Use docker compose to launch a PostgreSQL database:

```yaml
services:
  db:
    image: ankane/pgvector
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: talk_with_code
    ports:
      - '5432:5432'
    healthcheck:
      test: pg_isready -U postgres
      interval: 2s
      timeout: 3s
      retries: 40
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
```

4. Install `llama_index` in your virtual environment.

## Load the code from the repository

Load your `GITHUB_TOKEN` from the environment variable, choose a repository you want to load the code from.
See more details in <https://docs.llamaindex.ai/en/stable/examples/data_connectors/GithubRepositoryReaderDemo/>

```python
github_token = os.environ.get("GITHUB_TOKEN")
owner = "valeeraZ" # the owner of the repository
repo = "semantic-search" # the name of the repository
branch = "main" # the branch of the repository, or commit

from llama_index.readers.github.repository.github_client import GithubClient
from llama_index.readers.github import GithubRepositoryReader

client = GithubClient(github_token)

documents = GithubRepositoryReader(
    github_client=client,
    owner=owner,
    repo=repo,
    use_parser=False,
    verbose=False,
).load_data(branch=branch) # commit or branch
```

Use `len(documents)` we can see 41 documents are loaded from the repository.

Let's see the `Document` object:

```python
from IPython.display import JSON, display
display(documents[1].to_dict())
```

```
{'id_': '5d8c952370594f48a2b50a9654fcb979d46166c4',
 'embedding': None,
 'metadata': {'file_path': 'api/__main__.py',
  'file_name': '__main__.py',
  'url': 'https://github.com/valeeraZ/semantic-search/blob/main/api/__main__.py'},
 'excluded_embed_metadata_keys': [],
 'excluded_llm_metadata_keys': [],
 'relationships': {},
 'text': 'import logging\nimport os\nimport sys\n\nfrom asgi_correlation_id.context import correlation_id\nfrom loguru import logger\nfrom uvicorn import Config, Server\n\nfrom api.settings import settings\n\nLOG_LEVEL = logging.getLevelName(settings.log_level.value.upper())\nJSON_LOGS = True if os.environ.get("JSON_LOGS", "0") == "1" else False\n\n\ndef correlation_id_filter(record):\n    record["correlation_id"] = correlation_id.get()\n    return True\n\n\nclass InterceptHandler(logging.Handler):\n    def emit(self, record):\n        # get corresponding Loguru level if it exists\n        try:\n            level = logger.level(record.levelname).name\n        except ValueError:\n            level = record.levelno\n\n        # find caller from where originated the logged message\n        frame, depth = sys._getframe(6), 6\n        while frame and frame.f_code.co_filename == logging.__file__:\n            frame = frame.f_back\n            depth += 1\n\n        logger.opt(depth=depth, exception=record.exc_info).log(\n            level,\n            record.getMessage(),\n        )\n\n\ndef setup_logging():\n    # intercept everything at the root logger\n    logging.root.handlers = [InterceptHandler()]\n    logging.root.setLevel(LOG_LEVEL)\n\n    # remove every other logger\'s handlers\n    # and propagate to root logger\n    for name in logging.root.manager.loggerDict.keys():\n        logging.getLogger(name).handlers = []\n        logging.getLogger(name).propagate = True\n\n    # configure loguru\n    fmt = "<green>{time:YYYY-MM-DD HH:mm:ss.SSS}</green> | <level>{level: <8}</level> | <red> {correlation_id} </red> | <cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>"\n    logger.remove()\n    logger.add(\n        sink=sys.stdout,\n        format=fmt,\n        filter=correlation_id_filter,\n        serialize=JSON_LOGS,\n    )\n\n\ndef main() -> None:\n    """Entrypoint of the application."""\n    server = Server(\n        Config(\n            "api.web.application:get_app",\n            host=settings.host,\n            port=settings.port,\n            log_level=settings.log_level.value.lower(),\n            reload=settings.reload,\n            workers=settings.workers_count,\n        ),\n    )\n    setup_logging()\n    logger.info("Starting server...")\n    server.run()\n\n\nif __name__ == "__main__":\n    main()\n',
 'start_char_idx': None,
 'end_char_idx': None,
 'text_template': '{metadata_str}\n\n{content}',
 'metadata_template': '{key}: {value}',
 'metadata_seperator': '\n',
 'class_name': 'Document'}

```

The document object contains the text of the code and the metadata of the code file. Let's see what information we can get from the metadata:

- the URL of the code file on GitHub, and also the file path and file name.
- the text of the code file.

## Get the main programming language of each code file

In a large code base, we may have files with different programming/markup languages used in files. We will use the extension of the file to determine the language of the file.

I use a [JSON file](https://gist.github.com/ppisarczyk/43962d06686722d26d176fad46879d41#file-programming_languages_extensions-json-L13) to get the programming language by file extension name.

A JSON file like:

```json
[
    {
      "name":"Python",
      "type":"programming",
      "extensions":[
         ".py",
         ".bzl",
         ".cgi",
         ".fcgi",
         ".gyp",
         ".lmi",
         ".pyde",
         ".pyp",
         ".pyt",
         ".pyw",
         ".rpy",
         ".tac",
         ".wsgi",
         ".xpy"
      ]
   },
   {
      "name":"Java",
      "type":"programming",
      "extensions":[
         ".java"
      ]
   },
   ...
]
```

Given the file extension, we can get the programming language of the file.

```python

import json
f = open("languages.json", "r")
language_list = json.load(f)
f.close()

def get_name_by_extension(extension: str, language_list: list[dict]) -> str | None:
    for item in language_list:
        if extension.lower() in item.get("extensions", []):
            return item["name"].lower()
    return None

```

Now we have a function to get the programming language name by the file extension, we need to arrange the documents by their programming language.

```python
# group documents by file extension in file_path field in metadata
from llama_index.core.schema import Document
def arrange_documents_by_language(documents: list[Document]) -> dict:
    language_dict = {}
    for doc in documents:
        file_extension = "."+doc.metadata["file_path"].split(".")[-1]
        # get programming language name from file extension
        language_name = get_name_by_extension(file_extension, language_list)
        if language_name:
            if language_name in language_dict:
                language_dict[language_name].append(doc)
            else:
                language_dict[language_name] = [doc]
    return language_dict

file_extension_dict = arrange_documents_by_language(documents)
```

The `file_extension_dict` contains the documents grouped by their programming language.

```python
file_extension_dict["python"]

# [Document(_id='a'), Document(_id='b'), ...]

file_extension_dict.keys()
# dict_keys(['yaml', 'markdown', 'python', 'dockerfile', 'json', 'toml'])
```

## Start the RAG pipeline

### Prepare a vector store for embedding data

Embeddings created by embedding model are stored in a vector store that offers fast retrieval and similarity search by creating an index over our data. We will use the PostgreSQL database to store the embeddings.

```python
from sqlalchemy import make_url
from llama_index.core import SimpleDirectoryReader, StorageContext
from llama_index.core import VectorStoreIndex
from llama_index.vector_stores.postgres import PGVectorStore

connection_string = "postgresql://postgres:postgres@localhost:5432"
db_name = "talk_with_code"
url = make_url(connection_string)
vector_store = PGVectorStore.from_params(
    database=db_name,
    host=url.host,
    password=url.password,
    port="5432",
    user=url.username,
    table_name="valeeraz_github_repo", # the table name to store the embeddings, you can change it
    embed_dim=1536,  # openai embedding dimension
)
```

### Create embeddings for the code files

We can then create embeddings for Python documents and store them in the vector_store.

```python
from llama_index.core.node_parser import CodeSplitter
splitter = CodeSplitter(
    language="python"
)
index = VectorStoreIndex.from_documents(
    documents=file_extension_docs["python"],
    transformations=[splitter],
    storage_context=storage_context,
    show_progress=True,
)
```

We use the [`CodeSplitter`](https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/modules/#codesplitter) to split the code into smaller parts and make the embedding for each part.
We generated an `index` object that contains the embeddings of the Python files in the repository.

> See more details in the [llama index doc](https://docs.llamaindex.ai/en/stable/module_guides/indexing/vector_store_index/)

You can access the database and see the embeddings data is stored in the table `data_valeeraz_github_repo` in your PostgreSQL database.

### Query the index

Let's try to use this index, ask a question about a Python file in the repository.

```python

query_engine = index.as_query_engine(streaming=True, similarity_top_k=5) # search top 5 similar documents and generate response

response = query_engine.query('What is the lifetime of this application?')
print(response)
```

_The lifetime of this application involves running startup and shutdown events using a context manager defined in the `lifetime.py` file. The startup event involves setting up the database connection and creating tables, while the shutdown event disposes of the database engine. These events are managed by the `lifespan` context manager which ensures proper initialization and cleanup of resources when the FastAPI application starts and stops._

However, as this index doesn't contain all the file embeddings in the repository, ask a question like "What does the Dockerfile do in this repository?" will not return reasonable result.

```python
response = query_engine.query('What does the Dockerfile do in this repository?')
```

_The provided context information does not mention anything about a Dockerfile in the repository._

> For the above code, you can find documments with similar usage in the [llama index doc](https://docs.llamaindex.ai/en/stable/module_guides/indexing/vector_store_guide/)

### Multi programming languages files

As you can see in the part "Create embeddings for the code files", we can only create embeddings for Python files. We need to also create embeddings for all different type of files and store them in the vector store. We should be able to ask any question about the code in the repository within one index.

```python
from llama_index.core.node_parser import CodeSplitter

all_nodes = []
for language, docs in file_extension_docs.items():
    splitter = CodeSplitter(language=language)
    if splitter:
        print(f"Splitting {language} documents")
        try:
            nodes = splitter.get_nodes_from_documents(docs)
            all_nodes.extend(nodes)
        except Exception as e:
            print(f"Error in {language} documents: {e}")
    else:
        print(f"Skipping {language} documents")
```

65 nodes are created from 41 documents loaded above.

Use the nodes to create a new `VectorStoreIndex` to store the embeddings of all the files in the repository.

```python
from llama_index.core.node_parser import CodeSplitter
splitter = CodeSplitter(
    language="python"
)
index = VectorStoreIndex(nodes=all_nodes, storage_context=storage_context, vector_store=vector_store)
```

> See more details about using nodes to create a vector store index in the [llama index doc](https://docs.llamaindex.ai/en/stable/module_guides/indexing/vector_store_index/?h=vectorstoreindex#creating-and-managing-nodes-directly)

Now we can ask any question about the code in the repository.

```python
query_engine = index.as_query_engine(streaming=True, similarity_top_k=5) # search top 5 similar documents and generate response

response = query_engine.query('What does the Dockerfile do in this repository?')
print(response)
```

_The Dockerfile in this repository installs necessary dependencies, configures Poetry, copies the project requirements and application code, and then installs the project dependencies using Poetry. Finally, it sets the command to run the application._

## Have Chat Memory during the conversation

We want to store the memory of the chat, that's to say the LLM will answer questions with the context of the previous questions. We can use the `ChatMemory` to store the chat history.

> See more details about [ChatMemory](https://docs.llamaindex.ai/en/stable/module_guides/storing/chat_stores/?h=chatmemory#simplechatstore)

```python

from llama_index.core.tools import QueryEngineTool
from llama_index.core.agent import ReActAgent
from llama_index.core.storage.chat_store import SimpleChatStore
from llama_index.core.memory import ChatMemoryBuffer

from llama_index.core.agent import ReActAgent

chat_store = SimpleChatStore()

chat_memory = ChatMemoryBuffer.from_defaults(
    token_limit=3000,
    chat_store=chat_store,
    chat_store_key="user1",
)
query_engine_tools = QueryEngineTool.from_defaults(
        query_engine=query_engine,
    )

react_agent = ReActAgent.from_tools([query_engine_tools], verbose=True, memory=chat_memory)
```

In the above code, I use a memory chat store who store the chat conversation messages in the memory. I create an Agent to use the query engine and the memory to answer the questions.

> See more details about [ReActAgent](https://docs.llamaindex.ai/en/stable/examples/chat_engine/chat_engine_react/)

```python
react_agent.chat('In this application, what does the function split_text_into_chunks in api/api/web/service/file_chunk.py do?')
```

_Thought: The current language of the user is: English. I need to use a tool to help me answer the question.  
Action: query_engine_tool  
Action Input: {'input': 'What does the function split_text_into_chunks in api/api/web/service/file_chunk.py do?'}  
Observation: The function split_text_into_chunks in api/web/service/file_chunk.py splits a given text into chunks based on an ideal token size. It calculates the ideal size for each chunk, divides the text into chunks of that size, and returns a list of these chunks.  
Thought: I can answer without using any more tools. I'll use the user's language to answer  
Answer: The function `split_text_into_chunks` in `file_chunk.py` splits a given text into chunks based on an ideal token size. It calculates the ideal size for each chunk, divides the text into chunks of that size, and returns a list of these chunks._

```python
react_agent.chat("What is my last question?")
```

_Thought: (Implicit) I can answer without any more tools!  
Answer: Your last question was "In this application, what does the function split_text_into_chunks in api/api/web/service/file_chunk.py do?"_

You can actually see more details from the output of the code above, who shows that the agent has found relative documents about the question as RAG process.

## Conclusion

In this post, I have shown how to use `llama-index` to create embeddings for code files in a GitHub repository and use the embeddings to ask questions about the code in the repository. I also showed how to use the `ChatMemory` to store the chat history and use the history to answer the questions.

To improve:

- Use a Factory Pattern to get `CodeSplitter` for different programming languages for production.
- Learn more and use other Agent more adapted to our need about asking questons about the code in the repository or search more information from internet.
- Use a better OpenAI model to generate the response
