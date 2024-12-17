---
warning: DO NOT CHANGE THIS MANUALLY, THIS IS GENERATED BY https://github/duckdb/community-extensions repository, check README there
title: pyroscope
excerpt: |
  DuckDB Community Extensions
  DuckDB Pyroscope Extension for Continuous Profiling

extension:
  name: pyroscope
  description: DuckDB Pyroscope Extension for Continuous Profiling
  version: 0.0.1
  language: Rust
  build: cmake
  license: MIT
  excluded_platforms: "windows_amd64_rtools;windows_amd64;wasm_threads;wasm_eh;wasm_mvp"
  requires_toolchains: "rust;python3"
  maintainers:
    - lmangani
    - akvlad

repo:
  github: quackscience/duckdb-extension-pyroscope
  ref: 5b727e408b78cd725e6312f1bb8fd2aa84e7f8f8

docs:
  hello_world: |
    ---- Start the tracer, requires backend URL
    D SELECT * FROM trace_start('https://pyroscope:4000');
    
    ---- Stop the tracer
    D SELECT * FROM trace_stop();
    
  extended_description: |
    The Pyroscope Extension is experimental, use at your own risk!

extension_star_count: 11
extension_star_count_pretty: 11
extension_download_count: 175
extension_download_count_pretty: 175
image: '/images/community_extensions/social_preview/preview_community_extension_pyroscope.png'
layout: community_extension_doc
---

### Installing and Loading
```sql
INSTALL {{ page.extension.name }} FROM community;
LOAD {{ page.extension.name }};
```

{% if page.docs.hello_world %}
### Example
```sql
{{ page.docs.hello_world }}```
{% endif %}

{% if page.docs.extended_description %}
### About {{ page.extension.name }}
{{ page.docs.extended_description }}
{% endif %}

### Added Functions

<div class="extension_functions_table"></div>

| function_name | function_type | description | comment | example |
|---------------|---------------|-------------|---------|---------|
| trace_start   | table         |             |         |         |
| trace_stop    | table         |             |         |         |

