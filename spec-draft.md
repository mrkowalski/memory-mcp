## overview

An MCP server designed to remember specific bits of conversations with any genAI model, primarily used with Claude desktop.

## LLM interaction

A memory can by saved on an explicit user request or when LLM decides so. The MCP server must provide clear instructions to the LLM on it.

## technology

I want it to be a container so it will be easy to deploy on cloud and quickly movable between computers I use. Memories must be markdown files. I will version them with git. Each memory must be a separate markdown file with a front-matter.

- All code must by in Python 3.14. All must happen via `uv`. pytest with non-strict coverage target of 80%; ruff, mypy strict.
- MCP framework must be official SDK v2.
- Docker for containers. Use UID/GID from host.
- Git. Memory repo is a separate repo from the MCP memory server repo. MCP server does not manage the repo. It just knows where the directory is.
- No deployment as I will be running containers via local Docker / Docker compose.
- Scale: 500 memories per year.
- Language detection: fastText-predict. Fenced code, urls, markup, emails, file paths, etc. have to be stripped from the memory before language detection.
- All datetimes / timestamps have second precision.

### configuration

Env vars.

- port
- memories path
- log level
- log sink (file path, stdout)
- default DNF query
- max words returned for queries

### interface

MCP server must expose a HTTP interface. It will be used with Claude desktop via a stdio proxy; mcp-remote will be used for stdio proxy. HTTP, no auth. Plain JSON, stateless sessions.

#### MCP tools exposed

- create_memory
- update_memory
- delete_memory
- query_memories
- list_topics
- list_tags

## single memory structure

Each memory must have following front-matter attributes:

- 'created': datetime, UTC, ISO 8601; required
- 'topic': 2-3 words identifying a broad topic the memory belongs to. An LLM should try to use existing topics before introducing a new one; required
- 'tags': a list of tags. There can be at most 10 tags per memory. Max tag length is 32 characters. Tag length is at least 2 characters. Leading and trailing hyphens are not allowed. Consecutive hyphens are not allowed; required; empty list if no tags are set.
- 'uid': stable unique id; fatal error if not unique
- 'updated': uid of the memory being updated; optional
- 'deleted': datetime when deleted, UTC, ISO 8601; optional
- 'origin': user|llm; marks whether a memory creation was explicitly requested by a user or implicitly by LLM; required 

### memory body

Memories are always stored in English and are human readable. Memory language must be detected by the MCP server and rejected if it is not in English. Memories can be multi-line. Minimum memory body is 3 words. Word is detected by white-space split on raw markdown.

### memory size

Maximum memory length is 300 words; this limit applies to memory body only, frontmatter excluded. If memory length exceeds the limit, is must be rejected by the server. There is a configurable word limit for return queries that applies to the whole returned memory set, set to a 2000 words. Word limit is counted on memory bodies.

## persistence

### storage

Memories are saved as files on a machine local to the MCP server. GIT sync is applied manually. Each memory is a separate markdown file. Flat directory structure, no subfolders. All files that are not *.md must be ignored. Files without a frontmatter must be considered malformed. Subfolders must be ignored. Frontmatter is always the ground truth. Filenames are just for storage and orientation.

Memories are UTF-8 encoded, LF only. If memories need to be updated, only the affected parts can be changed. Memories not explicitly affected by the update must be preserved. Front-matter attribute order must be preserved.

All *.md files in a memory folder are considered. There is no exclusion list nor filename pattern matching.

#### filename

Filenames must be [date]-[time]-[uid].md 

Filename / frontmatter mismatch is ignored.

### uid

8-char base32 id; collision checked against existing memories. Crockford, canonical.

### reading

Memories are re-read fully on each memory request and there is no index. They are returned as a concatenated single block of text, filtered by a tag or tags.

#### request output

- <topic1>
  - <memory1.uid> <memory1.date> [#tag, #tag2]: <memory1.body>
 - <memory2.uid> <memory2.date> [#tag1, #tag2]: <memory2.body>
- <topic2>
  - ...

`date` is a full UTC ISO timestamp. Output is sorted by date, ascending, separately for each topic. Topics may not be sorted. Line breaks must be replaced by whitespace.

#### retrieval queries

Queries must accept a list of tags for filtering. It must be possible to filter by a combination of tags and AND / OR / NOT relationships between them using the DNF notation (Structured JSON). DNF queries must allow for querying for untagged memories. Each memory can have more than one tag or none. Matching must be exact, i.e. "hour" does not match "hours" nor "hour-mode".

I want an empty query to default to a configurable DNF query. Empty query means absent argument and also empty argument value, like empty list of tags. In default config, empty query defaults to memories that have a `default` tag. Invalid tags must be rejected (not matching 'tag format'). Tags are the only query attributes available. No other attributes can be queried.

In queries, tags must be lower case and match the tag format. Server does not normalize tags in queries.

#### tag format

Minus sign and digits are the only allowed non-ASCII-letter characters. Leading and trailing minus sign is not allowed. Leading digits are not allowed.

##### reserved tags

- `default`. It can be queried and is used in the default DNF query. It can never be set via MCP; it can only be read.
- `sensitive`. It can be queried and can be set via MCP on user or LLM discretion. LLM should report to a user that a saved memory was considered sensitive.

If the `default` tag appears in a create or update query, it is an error.

#### topic format

Topic must be 2 or 3 words, lowercase, ASCII, English, stored as words.

### error handling

As it is possible to edit memories manually, errors must be handled. Malformed frontmatter or any other issue with memory format / validation must result in an error-and-stop when server starts. Errors detected after server startup must result in server responses pointing to a list of faulty memory files until all errors are fixed. Single faulty memory means server is on but for all requests responds with information on which file is broken and does nothing else; same behavior for duplicate UIDs. Unknown front-matter attributes are ignored and preserved. Duplicate uids are fatal. Server-start rules are exactly the same as for re-validation on read.

Configurable confidence threshold for English language detection. Configurable min number of words for the language detection to be enabled. Defaults: 5 words, confidence threshold 0.7. If length is shorter than min number of words, it is accepted without language detection.

### concurrency

If conflicts stem from the fact of multiple clients being connected to the memory server, let them be. Race conditions are ok.

### duplicates

The server does not detect duplicates; there is no tool for it and it has no duplicate detection logic.

## single memory life cycle

- CRUD. It must be possible to create / read / update / delete a single memory per LLM request.
- **Create**
    - decided by the LLM. Date and time is taken from the server, LLM does not provide it. Topic must be provided and is preferably matched against a list of existing topics. It is ok to create a new topic and they can be merged afterwards manually. I think I will be using Obsidian for this.
- **Delete**; by uid. Delete is setting a deleted field to a deletion date. A system must accept physical manual deletion, too. If anything references an inexistent uid, it must be a warning at most. Deleted memories are never returned in queries and it is not possible to un-delete. Deleted means invisible in the interface. Deleting a deleted or inexistent memory is an error. Updating a deleted memory is an error.
- **Update**; by uid. Memory update is always a delete (set the `deleted` flag) + create a new one with 'updated' field set to the previous version. It is ok for a `updated` reference to point nowhere, in case old memories are deleted manually. Updated memories get their new `created` datetime. Updated memories (referenced in the `updated` field are effectively deleted. Update method must provide a list of what is to be updated and to what values. All fields that are not present in an update must be preserved.
- **Sensitive tag**: Meaning of sensitive: A user may opt out of saving any sensitive memories if an LLM deems them so. User may opt out of retrieving sensitive memories in a certain context. A user my want to verify what intimate memories are stored in the db while talking to an LLM. An LLM may decide itself on if a memory is sensitive and should report it to the user that it did. All is LLM enforced, no server side. LLM notifies if it applies or reads memories with a 'sensitive' tag.

## Requests that will be thrown against an LLM that this MCP must help to fulfill

- "What do you remember about <tag-query>", where X is a tag or a set of tags.
- "What are all topics memories belong to?"
- "Delete a memory with UID=x"
- "Create a memory <contents>"
- "Update a memory with UID=x with <contents>"
- "Give me a list of memories that are duplicates for <tag-query>.

## Initial version scope

- Cloud deployment is out of scope of the initial version. Will be added in later versions.
