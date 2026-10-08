# Keeping an Elasticsearch knowledge base in sync with Cognee

By Arya Gupta · 8 October 2026 · Mergetober integration demo

A team can keep onboarding guides and operational runbooks in Elasticsearch while
using Cognee to make those documents available to an assistant. The difficult part
is keeping memory current: an edited guide should replace its old content, and a
deleted guide should stop appearing in search.

This article walks through a local onboarding knowledge-base demonstration for
[Cognee issue #4770](https://github.com/topoteretes/cognee/issues/4770). The new
connector reads Elasticsearch with an API key, selects documents, and synchronizes
them into Cognee. The implementation is available on the
[contribution branch](https://github.com/aryagupta-tech/cognee-community/tree/feat/elasticsearch-connector/packages/connector/elasticsearch).
The contributor has reviewed the prepared contribution and authorized submission.
The upstream contribution is
[cognee-community PR #302](https://github.com/topoteretes/cognee-community/pull/302).

## The onboarding scenario

Start with a guide that says: “Request VPN access through the helpdesk.” Later, the
team switches to an identity portal. A useful assistant should retrieve the revised
instructions. If the guide is retired entirely, its old instructions should leave
Cognee's memory too.

The local demonstration runs a security-enabled Elasticsearch 8.17.10 instance,
an index-scoped API key, Cognee 1.6.3, and real SQLite, Ladybug graph and LanceDB
vector storage. Model and embedding calls use deterministic test doubles. That
makes the synchronization checks reproducible; evaluating natural-language answers
with a production model provider is a separate step.

## Set up the source

Use Python 3.11–3.13 and an Elasticsearch 8.x server. Clone the contribution and
install its connector package:

```sh
git clone --branch feat/elasticsearch-connector https://github.com/aryagupta-tech/cognee-community.git
cd cognee-community/packages/connector/elasticsearch
python -m venv .venv
source .venv/bin/activate
pip install -e '.[dev]'
```

The selected index needs `_source` enabled and an `updated_at` field mapped as
`date` or `date_nanos` with doc values. Each document needs one update date. A
minimal guide has this shape:

```json
{
  "title": "Team onboarding",
  "body": "Request VPN access through the helpdesk.",
  "updated_at": "2026-10-08T09:00:00Z",
  "published": true
}
```

Create an API key with `read` and `view_index_metadata` on all selected indices.
The metadata privilege lets the connector recognize an index recreated under the
same name. The ingestion key needs no write or cluster privileges. Use the
base64 `encoded` key from Elasticsearch's API-key response, and keep its document
and field permissions stable between syncs.

```sh
export ELASTICSEARCH_URL='https://your-elasticsearch:9200'
export ELASTICSEARCH_API_KEY='your-encoded-key'
export ELASTICSEARCH_INDEX='knowledge-*'
export ELASTICSEARCH_SOURCE_ID='knowledge-reader'
```

Configure Cognee's LLM and embedding providers for normal usage. For a private
Elasticsearch certificate authority, the example accepts `ELASTICSEARCH_CA_CERTS`;
certificate verification remains enabled. Detailed index mappings, API-key setup
and test commands are in the
[package README](https://github.com/aryagupta-tech/cognee-community/blob/feat/elasticsearch-connector/packages/connector/elasticsearch/README.md).

## Ingest only the knowledge you want

The source accepts an index or alias, an Elasticsearch query clause, and selected
JSON fields. This example includes published documents and leaves other fields
out of Cognee memory:

```python
import asyncio

import cognee
from cognee_community_connector_elasticsearch import elasticsearch_source


async def main():
    source = elasticsearch_source(
        source_id="knowledge-reader",
        index="knowledge-*",
        query={"term": {"published": True}},
        fields=["title", "body", "updated_at"],
    )
    await cognee.remember(
        source,
        dataset_name="onboarding",
        primary_key="id",
        write_disposition="merge",
        max_rows_per_table=0,
    )
    answers = await cognee.recall(
        "How do I request VPN access?", datasets=["onboarding"]
    )
    for answer in answers:
        print(answer)


asyncio.run(main())
```

Run the same source configuration and dataset on each sync. `merge` retains
unchanged staged documents while applying updates and deletion tombstones. Stable
IDs include the source configuration, concrete index and Elasticsearch document
ID, so identical IDs in different indices remain distinct.

The package also ships a runnable environment-based example:

```sh
export COGNEE_DATASET='onboarding'
export COGNEE_QUESTION='How do I request VPN access?'
python examples/remember_elasticsearch.py
```

That example selects all documents in the configured index. Use the source's
`query` and `fields` arguments, as above, when you need narrower selection.

## What changed during the verified demo

The automated example test executes the shipped example against a real
Elasticsearch API-key connection. It uses Cognee's actual `recall` API with chunk
retrieval selected for deterministic validation, rather than model-generated
answers. It checks relational records, graph text and vector document chunks.

| Source action | Verified Cognee result |
| --- | --- |
| Add the onboarding guide | One document/chunk; recall output includes the helpdesk instructions. |
| Sync again without an edit | The original record ID remains; one document/chunk. |
| Change the guide to the identity portal | One document/chunk; graph text contains the new instructions and omits the helpdesk text. |
| Delete the final guide and sync | No document records or document chunks; the retired guide's text is absent from the graph. |

These are local validation results recorded by an AI assistant on 8 October 2026.
They describe a reproducible onboarding use case with synthetic documents.

## Why pagination and deletion need different checks

The connector opens a point-in-time view and pages with `search_after` on the
update date plus Elasticsearch's `_shard_doc` tie-breaker. Each request carries
the entire returned sort tuple, and the connector closes the PIT after the scan.
This follows Elastic's
[PIT pagination guidance](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/paginate-search-results)
and supports result sets beyond the usual 10,000-hit offset limit.

An update cursor cannot reveal a hard deletion. Each sync therefore scans the
selected corpus's complete ID and revision inventory without document bodies.
Only new or changed bodies are fetched. Content fingerprints avoid re-ingesting
unchanged text; a successful full inventory identifies missing IDs for deletion.
The metadata scan is O(N), even when the content delta is small.

A failed request, partial shard result, missing index or incomplete content delta
aborts before publishing changes. Cognee's existing document cleanup removes
deleted records and their graph/vector artifacts. Cleanup failures in the core
are logged and retried on a later sync.

## Reproduce the checks

The suite includes offline lifecycle tests, live Elasticsearch tests with 10,017
documents across two shards, and real Cognee storage/recovery tests. Generated
sequences compare sync output with an independent upstream-state oracle.

The final combined run passed **183 tests**, plus **200 generated lifecycle
sequences**, with **100% connector statement and branch coverage**. Ruff lint,
formatting, dependency consistency and wheel packaging also passed. These results
cover the documented test environment and scenarios; they do not establish
correctness for every deployment.

A further graph regression checks extracted concepts, relationships and entity
vectors: a retired document's unique facts disappear, shared facts remain for the
surviving document, and deleting the last document leaves all tested stores empty.

For the combined example test, point these variables at a **disposable**,
security-enabled local Elasticsearch server. Test fixtures create and remove
uniquely named indices and API keys:

```sh
export ES_TEST_URL='http://127.0.0.1:19200'
export ES_TEST_PASSWORD='your-disposable-admin-password'
pytest tests/test_cognee_sync.py::test_runnable_example_with_live_source_updates_and_deletion -q
pytest tests -q --cov=cognee_community_connector_elasticsearch --cov-branch --hypothesis-show-statistics
```

The API key is generated at runtime and invalidated after the test. The admin
password is used for fixture setup and cleanup, while the connector authenticates
with the scoped reader key.

In deployment, keep read permissions stable, serialize syncs for the same source
and dataset, and retain both dlt state and its staging database. Different queries
or field selections create separate scopes. Production model quality and your
server's TLS configuration need validation in your own environment.

## Development disclosure

An AI assistant substantially prepared the implementation, tests and this article,
and ran the automated validation. Arya reviewed the prepared contribution and
authorized submission. Maintainer assignment is recorded on the issue;
Mergetober eligibility for this level of AI involvement has not been confirmed.
