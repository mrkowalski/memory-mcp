## overview

An MCP server designed to remember specific bits of converations with any genAI model, primarily used with Claude desktop.

## LLM interaction

A memory can by saved on an explicit user request or when LLM decides so. The MCP server must provide clear instructions to the LLM on it.

## technology

I want it to be a Docker container so it will be easy to deploy on cloud and quickly movable between computers I use. Memories must be markdown files. I will version them with git. Each memory must be a separate markdown file with a frontmatter.
All code must by in Python. All must happen via `uv`. 
Docker for containers.
Git.
No deployment as I will be running containers via local Docker / Docker compose.

### interface

MCP server must expose a HTTP interface. It will be used with Claude desktop via a stdio proxy.

## single memory structure

Each memory must have following frontmatter attributes:

- Date and time created; UTC, ISO 8601
- Topic. 2-3 words identifying a broad topic the memory belongs to. An LLM should try to use existing topics before introducing a new one.
- Client / model that registered the memory or information that it is human-sourced.
- stable unique id

Maximum memory length is 300 words. Memories are always stored in English. Human readable.

### storage

Each memory is a separage markdown file. Filenames must include a combination of [topic]-[title]-[date]-[time]

### persistence

Memories are save as files on a machine local to the MCP server. GIT sync is applied manually.

### concurrency

Please assume there is only one user of the memory server at the time. If there are conflicts, let them be. Race conditions are ok.

Memory retrieval must be based on tags. Request will ask for tags and all memories with a specific tag must be returned. If a memory has multiple tags, it is enough for it to contain any of the requested.

## single memory lifecycle

- new memory must be checked against an existing set of memories for duplicates.

## Requests that will be thrown against an LLM that this MCP must help to fulfill.

- "What memories have been saved in this conversation?"
- "What memories I have from this project?"
- "What do you remember about <X>", where X is a tag or a set of tags.
- "What are all topics memories belong to?"
