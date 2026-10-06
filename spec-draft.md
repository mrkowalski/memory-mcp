## overview

An MCP server designed to remember specific bits of converations with any genAI model, primarily used with Claude desktop.

## technology

I want it to be a Docker container so it will be easy to deploy on cloud and quickly movable between computers I use. Memories must be markdown files. I will version them with git. Each memory must be a separate markdown file with a frontmatter.
All code must by in Python. All must happen via `uv`. 
Docker for containers.
Git.
No deployment as I will be running containers via local Docker / Docker compose.

### interface

MCP server must expose an HTTP interface. It will be used with Claude desktop via a stdio proxy.

## single memory structure

Each memory must have following frontmatter attributes:

- Date and time created
- Project name that is a source of this memory (like Claude project)
- Chat unique identifier / name that is a source of this memory
- tag that must be never inferred and always explicitly provided by the user requesting a memory


Memory retrieval must be based on tags. Request will ask for tags and all memories with a specific tag must be returned. If a memory has multiple tags, it is enough for it to contain any of the requested.

Memory save must work in the following way:

- Fresh git pull; all conflicts resolved on the MCP server favour.
- git add, commit, push
- I want the repository to be cached on the MCP server so pull is fast.
- I want git push to happen asynchronously

Please assume there is only one user of the memory server at the time. If there are conflicts, let them be. Race conditions are ok.

The only human-facing interfaces to this solution are LLM chat via MCP and git repo on the other end.

## single memory lifecycle

- Memory must be written only when explicitly requested by a user. Its contents will be provided by an LLM, but the tool interface must make an LLM avoid creating memories automatically without an explicit user request.
- new memory must be checked against an existing set of memories for duplicates.

## Requests that will be thrown against an LLM that this MCP must help to fulfill.

- "What memories have been saved in this conversation?"
- "What memories I have from this project?"
- "What do you remember about <X>", where X is a tag or a set of tags.
