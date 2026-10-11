## overview

An MCP server designed to remember specific bits of converations with any genAI model, primarily used with Claude desktop.

## LLM interaction

A memory can by saved on an explicit user request or when LLM decides so. The MCP server must provide clear instructions to the LLM on it.

## technology

I want it to be a Docker container so it will be easy to deploy on cloud and quickly movable between computers I use. Memories must be markdown files. I will version them with git. Each memory must be a separate markdown file with a frontmatter.
All code must by in Python. All must happen via `uv`. 
Docker for containers.
Git. Memory repo is a separate repo from the MCP memory server repo. MCP server does not manage it. It just knows where the directory is.
No deployment as I will be running containers via local Docker / Docker compose.
Scale: 500 memories per year.

### interface

MCP server must expose a HTTP interface. It will be used with Claude desktop via a stdio proxy; mcp-remote will be used for stdio proxy. HTTP on localhost only, no auth.

## single memory structure

Each memory must have following frontmatter attributes:

- Date and time created; UTC, ISO 8601
- Topic. 2-3 words identifying a broad topic the memory belongs to. An LLM should try to use existing topics before introducing a new one.
- Client / model that registered the memory or information that it is human-sourced.
- stable unique id
- 'sensitive' flag for memories that are ... sensitive.

### memory body

Memories are always stored in English and are human readable. Memory langauge must be detected by the MCP server and rejected if it is not in English.

### memory size

Maximum memory length is 300 words; this limit applies to memory body only, frontmatter excluded. If memory lenght exceeds the limit, is must be rejected by the server.

## persistence

### storage

Memories are saved as files on a machine local to the MCP server. GIT sync is applied manually. Each memory is a separage markdown file. Filenames must include a combination of [topic]-[title]-[date]-[time]. Flat directory structure, no subfolders.

### reading

Memories are re-read fully on each memory request and there is no index. They are returned as a concatenated single block of text, filtered by a tag or tags.

#### request output

- <memory1.date> (<memory1.sensitive>): <memory1.body>
- <memory2.date> (<memory2.sensitive>): <memory2.body>

output is sorted by date, ascending.

### error handling

As it is possible to edit memories manually, errors must be handled. Malformed frontmatter or any issue with reading of a memory must result in an error-and-stop when server starts.

### concurrency

If conflicts stem from the fact of multiple clients being connected to the memory server, let them be. Race conditions are ok.

Memory retrieval must be based on tags. Request will ask for tags and all memories with a specific tag must be returned. If a memory has multiple tags, it is enough for it to contain any of the requested.

## single memory lifecycle

- CRUD. It must be possible to create / update / delete a single memory per LLM request.
- Create
    - decided by the LLM. Date and time is taken from the server, LLM does not provide it. Topic must be provided and is preferably matched against a list of existing topics.

## Requests that will be thrown against an LLM that this MCP must help to fulfill

- "What memories have been saved in this conversation?"
- "What memories I have from this project?"
- "What do you remember about <X>", where X is a tag or a set of tags.
- "What are all topics memories belong to?"
- "Delete a memory with UID=x"
- "Create a memory <contents>"
- "Update a memory with UID=x with <contents>"
