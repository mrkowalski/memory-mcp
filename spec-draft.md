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
- a list of tags
- Client from `clientinfo` that registered the memory.
- stable unique id
- 'sensitive' flag for memories that are ... sensitive.
- A project in which the memory was created. I.e a Claude Desktop project name

### memory body

Memories are always stored in English and are human readable. Memory langauge must be detected by the MCP server and rejected if it is not in English.

### memory size

Maximum memory length is 300 words; this limit applies to memory body only, frontmatter excluded. If memory lenght exceeds the limit, is must be rejected by the server.

## persistence

### storage

Memories are saved as files on a machine local to the MCP server. GIT sync is applied manually. Each memory is a separage markdown file. Filenames must include a combination of [topic]-[date]-[time]. Flat directory structure, no subfolders.

### reading

Memories are re-read fully on each memory request and there is no index. They are returned as a concatenated single block of text, filtered by a tag or tags.

#### request output

- <memory1.date> (<memory1.sensitive>) [#tag, #tag2]: <memory1.body>
- <memory2.date> (<memory2.sensitive>) [#tag1, #tag2]: <memory2.body>

output is sorted by date, ascending.

#### retrieval by tags

Queries must accept a list of tags for filtering. It must be possible to filter by a combination of tags and AND / OR relationships between them. For instance: (X AND Y) OR (Y AND Z). Each memory can have more than one tag or none. Tags are stored lower case and must be lower-cased on write. Matching must be exact, i.e. "hour" does not match "hours" nor "hour-mode". Minus sign and digits are the only allowed non-ascii-letter characters.

### error handling

As it is possible to edit memories manually, errors must be handled. Malformed frontmatter or any issue with reading of a memory must result in an error-and-stop when server starts.

### concurrency

If conflicts stem from the fact of multiple clients being connected to the memory server, let them be. Race conditions are ok.


## single memory lifecycle

- CRUD. It must be possible to create / update / delete a single memory per LLM request.
- Create
    - decided by the LLM. Date and time is taken from the server, LLM does not provide it. Topic must be provided and is preferably matched against a list of existing topics. It is ok to create a new topic and they can be merged afterwards manually. I think I will be using Obisidan for this.
- Delete; by uid. Delete is setting a deleted field to a deletion date. A system must accept physical manual deletion, too. If anything references an inexistend uid, it must be a warning at most.
- Update; by uid. Memory update is always a delete + create a new one with 'updated' field set to the previous version. It is ok for a `updated` reference to point nowhere, in case old memories are deleted manually.
- Sensitive flag: It must act as a tag, but the one with system meaning. It may even be a tag, if it does not interfere with other retrieval logic. Meaning of sensitive: A user may opt out of saving any sensitive memories if an LLM deems them so. User may opt out of retrieving sensitive memories is a certain context. A user my want to verify what intimate memories are stored in the db while talking to an LLM. An LLM may decide itself on if a memory is sensitive and should report it to the user that it did.

## Requests that will be thrown against an LLM that this MCP must help to fulfill

- "What memories I have from this project?"
- "What do you remember about <X>", where X is a tag or a set of tags.
- "What are all topics memories belong to?"
- "Delete a memory with UID=x"
- "Create a memory <contents>"
- "Update a memory with UID=x with <contents>"
- "Give me UIDs of memories that are duplicates for tag <tag-query>.

