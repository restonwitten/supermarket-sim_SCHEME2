# Retired Files

Files removed from the working tree because they no longer serve current
development. They are preserved in git history under the tag listed for each
batch. Nothing here should be used as a reference for current work.

## Retrieving a retired file

```bash
git show <tag>:<path>                 # print it
git checkout <tag> -- <path>          # restore it into the working tree
```

## 2026-09-30 — tag `retired/2026-09-30`

| Path                                                                         | Why retired                                                                                                                          | Superseded by                                                                                          |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| `dev/prompts/VP_XML_pulls_documentation_from_specification_descriptions.txt` | Prompt tooling for the Visual Paradigm XML model-source approach, which is retired.                                                  | DbC contract (`dev/contracts/`) as the sole model spec                                                 |
| `dev/prompts/VP_XML_pushes_documentation_to_specification_descriptions.txt`  | Same as above.                                                                                                                       | DbC contract                                                                                           |
| `dev/specification_descriptions.txt`                                         | Spec-description template (usage examples / counter-examples) that the VP_XML prompts pushed to and pulled from VP.                  | Usage examples and counter-examples in the DbC contract                                                |
| `data/store_layout.json`                                                     | Hierarchical space scaffold whose geometry was to come from a parsed diagram/model export — a retired pipeline. Not read by the app. | Output of the planned model data generator (Store layout built by the generator at app initialization) |
