### contradictions

1. `create_memory` and `update_memory` return status only, so the LLM never learns the new uid. It then can't report, reference or update what it just saved. Proposal: return status + uid.
2. `list_tags` and `list_topics` "do not fetch from deleted memories", but entries are "never removed, even at count 0". A tag used only by deleted memories has count 0 and stays listed. Which rule wins? Also, does the startup regeneration keep zero-count rows from the old CSV (merge), or rebuild from scratch, which would drop them?
3. Query tags must be lowercase, but the tag format allows uppercase letters. A stored uppercase tag could never be queried. Proposal: lowercase only everywhere, and reject uppercase on create and update.
4. `default` can't be sent via MCP. So an update that sends a tags list will strip a manually set `default`. Should the server carry `default` over automatically?
5. Startup is error-and-stop, but runtime is a degraded mode. In a container with a restart policy, stop-on-start means a crash loop, and Claude Desktop just sees a dead connection. Proposal: use the same degraded mode at startup. Or do you want to keep the hard stop?
6. Frontmatter uses `2026-09-23 17:46:07` with no zone marker, but output says "full UTC ISO timestamp". Should output use the identical string, or `2026-09-23T17:46:07Z`? Also, unquoted, YAML parses that value as a datetime object. Should it be stored as a quoted string?
7. "Only affected parts can be changed" sits awkwardly with update = delete + create. My reading:
   - The old file gets only `deleted` added and is otherwise byte-identical.
   - The new file copies the old frontmatter (order and unknown attributes) with the changes applied.
   
   Is that correct? And is `deleted` appended as the last attribute?

### update semantics

8. Which fields are updatable: body, topic, tags, origin? Is the new version's `origin` taken from the request or copied from the old one?
9. Are tags in an update a full replacement list, or add/remove operations?
10. The field name `updated` reads like a timestamp. Rename it to `supersedes`?
11. Is visibility decided solely by `deleted`? If someone manually removes `deleted` from a superseded memory, it reappears alongside its successor. Is that acceptable?
12. Is an update with no effective change an error or a no-op?

### validation

13. Which rules apply on read to manually edited files: language, 3-300 words, tag count and format, topic format, uid format? Proposal: enforce all structural rules on read, but run language detection only on write. A fastText false negative on read would take the whole server down.
14. Duplicate tags within one memory: reject or dedupe?
15. Is the 5-word language-detection threshold counted before or after stripping code, URLs and so on?
16. A 2-3 word topic is always below the detection threshold, so "English" is effectively LLM-enforced. Is the topic separator a single space? Letters only, so no digits or hyphens?
17. Where does the warning for a dangling `updated` reference surface: the log only, or tool responses too?

### query

18. When the 2000-word limit is hit, what gets dropped: whole memories, oldest or newest first? Should the response state that it was truncated and how many memories were omitted? This also affects the "find duplicates" use case.
19. DNF shape. Proposal: `{"any": [{"all": ["a","b"], "none": ["c"]}, {"untagged": true}]}`. Clauses inside `any` are ORed; within a clause, `all` tags are ANDed and `none` tags are excluded. Ok, or do you have your own schema?

### files and runtime

20. Where do the two CSVs live? Inside the memory directory they end up in the memory repo, even though they're regenerated at startup. Proposal: a separate path set by an env var.
21. Should memory files and CSVs be written atomically (temp file + rename)? Races are accepted, but a half-written file would trigger the broken-file mode.
22. Which fastText model: `lid.176.ftz` (~1 MB, slightly less accurate) or `lid.176.bin` (~126 MB)? Baked into the image at build time?
23. If `fasttext-predict` has no cp314 wheel, is compiling it in a multi-stage build acceptable?
24. Should compose publish the port on `127.0.0.1` only? With no auth, `0.0.0.0` exposes the server to your LAN.
25. How many rotated logs should be kept?
26. Do you want a healthcheck endpoint for compose?

### LLM behaviour

27. The server can't force "always retrieve", the save triggers or the sensitive-tag reporting. It can only convey them through the MCP `instructions` field and tool descriptions. Should the server ship these instructions, possibly configurable by env var, or will you put them in Claude Desktop's project or system prompt?
