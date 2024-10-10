# platformOS OpenAI Module

The goal of this module is to provide an integrating with [OpenAI Embeddings API](https://platform.openai.com/docs/guides/embeddings) and provide [Commands](https://github.com/Platform-OS/pos-module-core?tab=readme-ov-file#commands--business-logic) and [Queries](https://github.com/Platform-OS/pos-module-core?tab=readme-ov-file#queries--accessing-data) for CRUD manipulation in [platformOS Embeddings backend](https://documentation.platformos.com/developer-guide/embeddings/embeddings)

## Installation

The platformOS OpenAI Module is available on the [Partner Portal Modules Marketplace](https://partners.platformos.com/marketplace/pos_modules/143).

### Prerequisites

Before installing the module, ensure that you have [pos-cli](https://github.com/mdyd-dev/pos-cli#overview) installed. This tool is essential for managing and deploying platformOS projects.

The platformOS OpenAI Module is fully compatible with [platformOS Check](https://github.com/Platform-OS/platformos-lsp#platformos-check----a-linter-for-platformos), a linter and language server that supports any IDE with Language Server Protocol (LSP) integration. For Visual Studio Code users, you can enhance your development experience by installing the [VSCode platformOS Check Extension](https://marketplace.visualstudio.com/items?itemName=platformOS.platformos-check-vscode).

### Installation Steps

1. **Navigate to your project directory** where you want to install the OpenAI Module.

2. **Run the installation command**:

```bash
   pos-cli modules install openai
```
This command installs the OpenAI Module along with its dependencies ([pos-module-core()](https://github.com/Platform-OS/pos-module-core)) and updates or creates the `app/pos-modules.json` file in your project directory to track module configurations.

### Setup

1. [Create an OpenAI Api Key](https://platform.openai.com/docs/quickstart/step-2-set-up-your-api-key)
2. Setup `modules/openai/OPENAI_SECRET_TOKEN` [Constant](https://documentation.platformos.com/api-reference/liquid/platformos-objects#context-constants) and provide the generated API key as a value

### Pulling the Source Code

To use advanced features such as autocomplete for `function`, `include`, and `graphql` tags, you'll need to include the module's source code in your project. Follow these steps:

1. **Pull the Source Code**: Use the `pos-cli` command to pull the source code into your local environment:

```
pos-cli modules pull openai
```

2. **Update Your .gitignore**: Add `modules/openai` to your `.gitignore` file to avoid directly modifying the module files. This precaution helps maintain the integrity of the module and simplifies future updates. If needed, repeat this step for every dependency, like for example the core module.

#### Managing Module Files

The default behavior of modules is that **the files are never deleted**. It is assumed that developers might not have access to all of the files, and thanks to this feature, they can still overwrite some of the module's files without breaking them. Since the OpenAI Module is fully public, it is recommended to delete files on deployment. To do this, ensure your `app/config.yml` includes the OpenAI Module in the list `modules_that_allow_delete_on_deploy`:

``` yaml
modules_that_allow_delete_on_deploy:
- openai
```

## Functionality provided by the user module:

- [x] **[Api Call to OpenAI Embeddings API](#api-call-to-openai-embeddings-api)**:  
- [x] **[Embeddings CRUD operations](#embeddings-crud-operations)**

### Api Call to OpenAI Embeddings API

The main module functionality is triggering an API Call to https://api.openai.com/v1/embeddings through the `modules/openai/openai/fetch_embeddings` defined in `modules/openai/public/lib/commands/openai/fetch_embeddings.liquid`. It automatically generates authorization header based on the`modules/openai/OPENAI_SECRET_TOKEN` constant value. .

The command takes `object` as an input, which is a string or an array of strings that you would like to transform into embedding(s).
The command returns OpenAI Embeddings API response, which will be a json.

> [!NOTE] 
> The module uses `text-embedding-ada-002` model, as platformOS expects embeddings of length 1536. 

#### Examples

##### transforming a string into an embedding:

{% assign object = "hello world" %}
{% function response = 'modules/openai/commands/openai/fetch_embeddings', object: object %}

##### transforming an array of strings into embeddings:

{% assign object = "hello world,this can be a long text" | split: ',' %}
{% function response = 'modules/openai/commands/openai/fetch_embeddings', object: object %}

### Embeddings CRUD operations

There are four additional commands provided by the module to Create Read Update and Delete platformOS Embeddings

#### Create
To create a new embedding returned by the OpenAI Embeddings API, you should use `modules/openai/embeddings/create` command:

```
assign pos_embedding_input = '{}' | parse_json
assign metadata = '{}' | parse_json

hash_assign metadata['title'] = "A title" # metadata allows us to cache any relevant metadata to avoid performing any extra DB query. 
hash_assign pos_embedding_input['metadata'] = metadata
hash_assign pos_embedding_input['embedding'] = response.data.first.embedding # assumes the response is the result of fetch_embeddings command and we invoked it for one string, which is why we are interested only in the first embedding. This field should contain embedding of length 1536, which is what OpenAI Embedding API returns for text-embedding-ada-002 model.
hash_assign pos_embedding_input['content'] = "Hello world" # usually we want to cache the original input that we transformed into an embedding.
function pos_embedding = 'modules/openai/commands/embeddings/create', object: pos_embedding_input
```

#### Update
Updating existing embedding can be achieved with `modules/openai/embeddings/update`. The input is the same as the one for the create, with one addition - you have to specify the `id` of the embedding which you would like to update:

```
{% liquid
  # ...
  hash_assign pos_embedding_input['id'] = embedding.id
  function pos_embedding = 'modules/openai/commands/embeddings/update', object: pos_embedding_input
%}
```

#### Delete 

You can also delete an existing embedding by invoking `modules/openai/embeddings/delete` command:

```
{% function pos_embedding = 'modules/openai/commands/embeddings/delete', id: embedding.id %}
```

#### Read

To leverage embeddings, at some point you would need to perform search operation through `modules/openai/queries/embeddings/search.liquid`. Usually you would like to sort the results by relevancy to the embedding that is provided as an input (for example, user enters their search query, which you transform into an embedding using `fetch_embedding` command, and you want to get 5 most relevant embeddings):

```
  function related_embeddings = 'modules/openai/queries/embeddings/search', related_to: embedding, limit: 5
```

## Example application

One of the typical use case of using Embedding is to implement AI enhanced search. This is also the first step of RAG (Retrieval-Augmented Generation). You can find an example implementation of such functionality in the `app` directory, ready to be copied and customized for your application needs, or further integration with LLM of your choice. The example shows example implementation of key concepts:

### A script to transform existing data into Embeddings

You can find the script in `app/lib/commands/openai/commands_to_embeddings.liquid`. It fetches the existing [Pages](https://documentation.platformos.com/developer-guide/pages/pages), filters them by your needs, and creates embeddings based on their content. This is meant for demonstration purposes - instead of creating embeddings for the whole page, you might want for example to create embeddings for certain paragraphs, which probably would yield a better results.

The script can be run multiple times and will remove embeddings of pages that have been removed since the last run, will update the pages that have changed, and finally will add new embeddings for new pages.

the script assumes that each Page that is index has configured [Metadata](https://documentation.platformos.com/developer-guide/pages/metadata) - `title` and `description`. Those will be required to efficiently render [Search Page](#search-page) without having to make another query to the DB.

### Search page

The example search page is located at `app/views/pages/openai_search.liquid`. It is just for demonstration purposes, and it violates some of the [best practices](https://documentation.platformos.com/developer-guide/modules/platformos-modules#how-to-use-devkit) like separating business rules from the presentation layer. This is intentional, as the page should just serve as an all-in-one example of the whole flow. 

You will find there additional checks if the module [setup](#setup) has been done correctly, it supports hcaptcha challenge to avoid spam requests and demonstrates how to use the [commands and queries](#functionality-provided-by-the-user-module) provided by the OpenAI module.

### Further considerations

What is not mentioned in the application, is ensuring that your embeddings are up to date. You can achieve it by either invoking [the script](#a-script-to-transform-existing-data-into-embeddings) regularly, or, preferably, by leveraging [Events](https://github.com/Platform-OS/pos-module-core?tab=readme-ov-file#events) and create consumers that will update Embeddings whenever an entity that you would like to track is created/updated/deleted.  

## Contribution

Please check `.git/CONTRIBUTING.md`
