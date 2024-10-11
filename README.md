# platformOS OpenAI Module

The platformOS OpenAI Module integrates the [OpenAI Embeddings API](https://platform.openai.com/docs/guides/embeddings) to enable CRUD operations on platformOS Embeddings, providing [Commands](https://github.com/Platform-OS/pos-module-core?tab=readme-ov-file#commands--business-logic) and [Queries](https://github.com/Platform-OS/pos-module-core?tab=readme-ov-file#queries--accessing-data) to manage the data lifecycle within [platformOS Embeddings backend](https://documentation.platformos.com/developer-guide/embeddings/embeddings).

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
This command installs the OpenAI Module along with its dependencies, such as [pos-module-core](https://github.com/Platform-OS/pos-module-core), and updates or creates the `app/pos-modules.json` file in your project directory to track module configurations.

### Setup

1. **Create an OpenAI API Key**: Follow the official OpenAI instructions to [create an OpenAI API Key](https://platform.openai.com/docs/quickstart/step-2-set-up-your-api-key).
2. **Configure the `OPENAI_SECRET_TOKEN` [Constant](https://documentation.platformos.com/api-reference/liquid/platformos-objects#context-constants)**: Store your API key in the `modules/openai/OPENAI_SECRET_TOKEN` as a constant for secure access.

### Pulling the Source Code

To use advanced features such as autocomplete for `function`, `include`, and `graphql` tags, you'll need to include the module's source code in your project. Follow these steps:

1. **Pull the Source Code**: Use the `pos-cli` command to pull the source code into your local environment:

```
pos-cli modules pull openai
```

2. **Update Your .gitignore**: Add `modules/openai` to your `.gitignore` file to avoid directly modifying the module files. This precaution helps maintain the integrity of the module and simplifies future updates. If needed, repeat this step for every dependency, such as the core module.

#### Managing Module Files

The default behavior of modules is that **the files are never deleted**. It is assumed that developers might not have access to all of the files, and thanks to this feature, they can still overwrite some of the module's files without breaking them. Since the OpenAI Module is fully public, it is recommended to delete files on deployment. To do this, ensure your `app/config.yml` includes the OpenAI Module in the list `modules_that_allow_delete_on_deploy`:

``` yaml
modules_that_allow_delete_on_deploy:
- openai
```

## Functionality provided by the OpenAI module:

- [x] **[API Call to OpenAI Embeddings API](#api-call-to-openai-embeddings-api)**:  
- [x] **[Embeddings CRUD Operations](#embeddings-crud-operations)**

### API Call to OpenAI Embeddings API

This module function triggers an API call to `https://api.openai.com/v1/embeddings` through the `modules/openai/openai/fetch_embeddings` defined in `modules/openai/public/lib/commands/openai/fetch_embeddings.liquid`. It automatically generates an authorization header using the `modules/openai/OPENAI_SECRET_TOKEN` constant.

The command takes an `object` as an input, which is a string or an array of strings that you would like to transform into embedding(s).
The command returns a JSON response containing the embeddings.

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

This section describes additional commands provided by the module to Create, Read, Update, and Delete platformOS Embeddings.

#### Create

To create a new embedding returned by the OpenAI Embeddings API, use the `modules/openai/embeddings/create` command as shown below:

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
Update existing embeddings using the `modules/openai/embeddings/update` command. The input is the same as the one for the create, with one addition - you have to specify the `id` of the embedding which you would like to update:

```
{% liquid
  # ...
  hash_assign pos_embedding_input['id'] = embedding.id
  function pos_embedding = 'modules/openai/commands/embeddings/update', object: pos_embedding_input
%}
```

#### Delete 

Delete an existing embedding with the `modules/openai/embeddings/delete` command:

```
{% function pos_embedding = 'modules/openai/commands/embeddings/delete', id: embedding.id %}
```

#### Read

To leverage embeddings, you will need to perform a search operation through `modules/openai/queries/embeddings/search.liquid`. Typically, you'll want to sort the results by their relevance to the input embedding (for example, when a user enters a search query, which you transform into an embedding using the `fetch_embedding` command, and you want to retrieve the five most relevant embeddings):

```
  function related_embeddings = 'modules/openai/queries/embeddings/search', related_to: embedding, limit: 5
```

## Example application

One typical use case for embeddings is to implement AI-enhanced search, which is also the first step in Retrieval-Augmented Generation (RAG). You can find an example implementation of this functionality in the `app` directory. It’s ready to be copied and customized for your application needs or further integrated with the LLM of your choice. This example demonstrates key concepts in action.

### Script to Transform Existing Data into Embeddings

You can find the script in `app/lib/commands/openai/commands_to_embeddings.liquid`. It fetches the existing [Pages](https://documentation.platformos.com/developer-guide/pages/pages), filters them according to your needs, and creates embeddings based on their content. This is intended for demonstration purposes; for instance, instead of creating embeddings for entire pages, you might choose to create embeddings for specific paragraphs, which could yield better results.

The script can be run multiple times and will remove embeddings of pages that have been deleted since the last run, update embeddings for pages that have changed, and add new embeddings for newly added pages.

The script assumes that each indexed page has configured [Metadata](https://documentation.platformos.com/developer-guide/pages/metadata) — `title` and `description`. These are required to efficiently render the [Search Page](#search-page) without needing to make another database query.

### Search page

The example search page is located at `app/views/pages/openai_search.liquid`. It serves primarily for demonstration purposes and intentionally violates some [best practices](https://documentation.platformos.com/developer-guide/modules/platformos-modules#how-to-use-devkit), such as separating business rules from the presentation layer. This design choice aims to present an all-in-one example of the entire workflow.

You'll find there additional checks to ensure the module [setup](#setup) has been configured correctly. It supports the hCaptcha challenge to prevent spam requests and demonstrates how to use the [commands and queries](#functionality-provided-by-the-user-module) provided by the OpenAI module.

### Further Considerations

**Maintaining Updated Embeddings**:  An important aspect not covered in the application is ensuring that your embeddings remain up-to-date. You can achieve this by regularly invoking [the script](#a-script-to-transform-existing-data-into-embeddings) designed to refresh embeddings. Alternatively, and preferably, you can leverage [Events](https://github.com/Platform-OS/pos-module-core?tab=readme-ov-file#events) to create consumers that automatically update embeddings whenever an entity you wish to track is created, updated, or deleted.

## Contribution

For contributing to this module, please check `.git/CONTRIBUTING.md`.
