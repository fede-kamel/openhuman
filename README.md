# OCI Generative AI in OpenHuman: end-to-end verification

The chain under test:

1. **tinyinference#122**: OCI Generative AI chat provider (`tinyhumansai/tinyinference`, head `b9b6830`)
2. **tinyagents**: OpenHuman's current pin, with its vendored tinyinference moved to #122's head (stands in for the re-cut tinyagents#517)
3. **openhuman#8183**: OCI as a cloud provider, **rebased onto current `main`**:
   branch [`fede-kamel/openhuman:oci-genai-rebased`](https://github.com/fede-kamel/openhuman/tree/oci-genai-rebased)
   (the original 6 commits, then 2 commits that adapt them to `main`)

Everything ran offline against a local stand-in for the OCI inference service. The stand-in verifies
each request as OCI does: it rebuilds the signing string from the `headers=` list, checks the
RSA-SHA256 signature against the registered public key, and checks `x-content-sha256` and
`content-length` against the body. The signing key and OCI profile were generated for the run and
use placeholder tenancy and user OCIDs. No Oracle Cloud account or credentials were involved.

| # | What | Result |
|---|------|--------|
| 1 | tinyinference#122: `fmt`, `clippy -D warnings`, `build`, `test --all-features`, `doc -D warnings` | all pass, 1,633 tests, 87 OCI tests |
| 2 | tinyinference#122 offline demo: unary, SSE streaming, streamed tool round trip, wrong-key rejection | 4/4 |
| 3 | openhuman#8183 (rebased): OCI unit tests | 38/38 |
| 4 | A real OpenHuman agent (`openhuman-embed`) routed `oci:meta.llama-3.3-70b-instruct` through a `[[cloud_providers]]` entry with `auth_style = "oci"`, `auth_mode = "config_profile"`, `transport = "native_generic"`, and a `get_weather` host tool | tool call, tool run, final answer |
| 5 | What the OCI stand-in received and sent for run 4 | 2 signed requests verified, SSE replies |

## Rebasing #8183 onto `main`

- **Conflicts.** 26 files conflicted. The i18n files kept both sides. The Cargo feature lists kept
  `oci-principals` and dropped the stale `tinymemes` context, because `main` had removed that
  feature. Code that `main` moved or deleted took `main`'s version.
- **`oci: None` in moved code.** It went into the `CloudProviderCreds` literals that `main` moved
  into new modules: `model_byok_route.rs`, `cloud_providers.rs` and two test files.
- **Async functions.** `main` made `route_has_usable_credentials`, the cloud-slug factory and
  `client_config_json` async. The keyless-OCI check was kept in front of the awaited key lookup,
  and the OCI tests now run under `#[tokio::test]` on `inference_test_guard_async()`.
- **Inspection credential.** Inspection now takes a pre-resolved credential. For a keyless OCI
  profile, `lookup_key_for_slug_read_only` returns an empty key and `entry_requires_stored_key`
  lets it through. The test passes the same empty credential.

## Caveats

- The stand-in's model is scripted. It shows the wiring (config, signing, wire format, streaming,
  tool calls) but not model quality. `crates/tinyinference-llm/examples/live_oci.rs` in #122 runs
  the same checks against the real service when OCI credentials are available.
- **Demo-only settings.** The demo agent turns off the prompt screen, because the TinySecurity
  module did not load on the offline machine, and it builds `openhuman-embed` with `storage-sqlite`.
  Neither setting touches the OCI path.
- **The pushed branch still pins `main`'s `vendor/tinyagents`.** It needs the tinyagents release
  that includes #122 before CI can build it, which is the merge order the maintainers set.

## 4-openhuman-agent-on-oci

```text
OpenHuman agent → OCI Generative AI
provider oci · auth config_profile (DEMO) · transport native_generic · route oci:meta.llama-3.3-70b-instruct

you ▸ What's the weather in Paris?
[iteration 1]
tool ▸ get_weather({"city":"Paris"})
tool ◂ get_weather success=true 626ms → {"sky":"cloudy","temperature_c":17}
[iteration 2]
agent ▸ It is 17°C and cloudy in Paris right now, according to get_weather.

✔ turn finished reply: "It is 17°C and cloudy in Paris right now, according to get_weather."
```

## 5-oci-service-wire-log

```text
OCI Generative AI stand-in on http://127.0.0.1:18443
accepting keyId ocid1.tenancy.oc1..demotenancy/ocid1.user.oc1..demouser/b8:a0:8f:8c:ef:73:35:3d:36:0c:53:2b:20:a9:de:72
ready

→ #1 POST /20231130/actions/chat (820 bytes)
  ✔ signature verified keyId=ocid1.tenancy.oc1..demotenancy/ocid1.user.oc1..demouser/b8:a0:8f:8c:ef:73:35:3d:36:0c:53:2b:20:a9:de:72
  ✔ body sha256 + length match signed: (request-target) date host x-content-sha256 content-type content-length
  compartmentId ocid1.compartment.oc1..democompartment
  servingMode   ON_DEMAND meta.llama-3.3-70b-instruct
  apiFormat     GENERIC · isStream=true · messages [SYSTEM, USER]
  tools         get_weather
  last USER   "Current Date & Time: 2026-11-25 19:13:14 Etc/UTC (+00:00, UTC+00:00), Wednesday What's the weather i…"
  ← 200 text/event-stream · tool call get_weather({"city":"Paris"}) id=call_7f3a
    sse ▸ data: {"index":0,"message":{"role":"ASSISTANT","toolCalls":[{"arguments":"{\"city\":","id":"call_7f3a","name":"get_weather","type":"FUNCTION"}]}}
    sse ▸ data: {"index":0,"message":{"role":"ASSISTANT","toolCalls":[{"arguments":"\"Paris\"}","type":"FUNCTION"}]}}
    sse ▸ data: {"finishReason":"tool_calls","index":0}
    sse ▸ data: {"usage":{"completionTokens":18,"promptTokens":1200,"totalTokens":1218}}
    sse ▸ data: [DONE]

→ #2 POST /20231130/actions/chat (1068 bytes)
  ✔ signature verified keyId=ocid1.tenancy.oc1..demotenancy/ocid1.user.oc1..demouser/b8:a0:8f:8c:ef:73:35:3d:36:0c:53:2b:20:a9:de:72
  ✔ body sha256 + length match signed: (request-target) date host x-content-sha256 content-type content-length
  compartmentId ocid1.compartment.oc1..democompartment
  servingMode   ON_DEMAND meta.llama-3.3-70b-instruct
  apiFormat     GENERIC · isStream=true · messages [SYSTEM, USER, ASSISTANT, TOOL]
  tools         get_weather
  last TOOL   "{\"sky\":\"cloudy\",\"temperature_c\":17}"
  ← 200 text/event-stream · text "It is 17°C and cloudy in Paris right now, according to get_weather."
    sse ▸ data: {"index":0,"message":{"content":[{"text":"It ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"is ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"17°C ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"and ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"cloudy ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"in ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"Paris ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"right ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"now, ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"according ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"to ","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"index":0,"message":{"content":[{"text":"get_weather.","type":"TEXT"}],"role":"ASSISTANT"}}
    sse ▸ data: {"finishReason":"stop","index":0}
    sse ▸ data: {"usage":{"completionTokens":18,"promptTokens":1200,"totalTokens":1218}}
    sse ▸ data: [DONE]
```

## 3-openhuman-8183-oci-tests

```text
# openhuman#8183 rebased on current main (branch fede-kamel/openhuman:oci-genai-rebased) — OCI tests
# vendor/tinyagents/vendor/tinyinference checked out at tinyinference#122 head (b9b6830)

test config::schema::cloud_providers::tests::auth_style_parse_accepts_oci_and_the_existing_styles ... ok
test config::schema::cloud_providers::tests::auth_style_parse_error_lists_oci_among_the_valid_values ... ok
test config::schema::oci_settings::tests::defaults_are_api_key_over_openai ... ok
test config::schema::oci_settings::tests::only_profile_modes_read_the_config_file ... ok
test config::schema::oci_settings::tests::legacy_entries_without_oci_deserialize_to_none ... ok
test config::schema::cloud_providers::tests::oci_provider_entry_survives_the_rpc_payload_shape ... ok
test inference::provider::factory::cloud_spec::tests::oci_has_no_compatible_registry_auth_style ... ok
test config::schema::oci_settings::tests::wire_names_round_trip_for_every_variant ... ok
test config::schema::cloud_providers::tests::documented_toml_oci_entries_deserialize ... ok
test config::schema::oci_settings::tests::provider_entry_round_trips_with_oci_settings ... ok
test config::schema::cloud_providers::tests::oci_builtin_preset_maps_to_the_oci_auth_style ... ok
test inference::provider::factory::oci::tests::api_key_mode_requires_a_region ... ok
test inference::provider::factory::oci::tests::blank_entry_endpoint_keeps_the_region_host ... ok
test inference::provider::factory::oci::tests::api_key_mode_without_a_key_names_the_problem ... ok
test inference::provider::factory::oci::tests::entry_endpoint_overrides_the_region_host ... ok
test inference::provider::factory::oci::tests::home_relative_config_paths_expand_to_the_home_directory ... ok
test inference::provider::factory::oci::tests::errors_never_contain_key_material ... ok
test inference::provider::factory::oci::tests::explicit_region_overrides_the_profile_region ... ok
test inference::provider::factory::oci::tests::config_profile_reads_the_profile_and_inherits_its_region ... ok
test inference::provider::factory::oci::tests::native_and_actions_transports_require_a_compartment ... ok
test inference::provider::factory::oci::tests::missing_profile_and_missing_file_are_reported_with_the_provider_slug ... ok
test inference::provider::factory::oci::tests::principal_modes_are_unsupported_without_the_feature ... ok
test inference::provider::factory::oci::tests::project_and_dedicated_endpoint_are_carried_into_the_config ... ok
test config::ops::tests::loader_and_search_tests::client_config_json_exposes_oci_settings_only_on_oci_entries ... ok
test inference::provider::factory::oci::tests::profile_kind_must_match_the_auth_mode ... ok
test inference::provider::factory::oci::tests::stored_key_is_required_only_for_api_key_mode ... ok
test inference::provider::factory::oci::tests::the_builtin_preset_endpoint_keeps_the_region_host ... ok
test inference::provider::factory::oci::tests::signed_auth_on_the_openai_transport_requires_a_compartment ... ok
test inference::provider::factory::oci::tests::the_openai_transport_requires_a_project ... ok
test inference::provider::factory::oci::tests::the_route_temperature_override_reaches_the_config ... ok
test inference::provider::factory::routing::routing_tests::keyless_oci_routes_count_as_usable_credentials ... ok
test inference::provider::models::catalog_listing_tests::oci_entries_answer_with_an_empty_catalog_without_a_network_request ... ok
test inference::provider::factory::oci::tests::entry_endpoint_override_must_be_https_or_loopback ... ok
test inference::provider::factory::oci::tests::inspection_records_the_oci_wire_identity ... ok
test inference::provider::factory::oci::tests::session_token_mode_uses_the_token_file_profile ... ok
test inference::provider::factory::oci::tests::oci_slug_routes_through_the_cloud_slug_factory ... ok
test inference::provider::factory::oci::tests::oci_slug_with_a_missing_compartment_fails_through_the_factory ... ok
test inference::provider::factory::oci::tests::api_key_mode_builds_over_each_transport ... ok
test result: ok. 38 passed; 0 failed; 0 ignored; 0 measured; 10833 filtered out; finished in 0.06s
```

## 2-tinyinference-122-offline-demo

```text
OCI Generative AI provider · offline end-to-end demo
stand-in service at http://127.0.0.1:37179 · model meta.llama-3.3-70b-instruct · throwaway 2048-bit RSA key

1. Plain completion (unary)
    [oci service] POST /20231130/actions/chat · signature verified · body sha256 ok · signed headers [(request-target) date host x-content-sha256 content-type content-length]
    model ▸ pong
    ✔ PASS text "pong", usage in=42 out=9 total=51, finish=Some("stop")

2. Streamed completion (server-sent events)
    [oci service] POST /20231130/actions/chat · signature verified · body sha256 ok · signed headers [(request-target) date host x-content-sha256 content-type content-length]
    model ▸ 1, 2, 3, 4, 5
    ✔ PASS reassembled "1, 2, 3, 4, 5", finish=Some("stop"), usage reported

3. Streamed tool call → local tool → final answer
    [oci service] POST /20231130/actions/chat · signature verified · body sha256 ok · signed headers [(request-target) date host x-content-sha256 content-type content-length]
    tool-call fragment id="call_7f3a" name="get_weather" args+="{\"city\":"
    tool-call fragment id="call_7f3a" name="get_weather" args+=" \"Paris\"}"
    assembled call call_7f3a get_weather({"city":"Paris"})
    running get_weather locally → {"temperature_c": 17, "sky": "cloudy"}
    [oci service] POST /20231130/actions/chat · signature verified · body sha256 ok · signed headers [(request-target) date host x-content-sha256 content-type content-length]
    model ▸ It is 17°C and cloudy in Paris right now.
    ✔ PASS call id kept across fragments, args {"city":"Paris"}, answer uses the tool result

4. Request signed with the wrong key is rejected
    [oci service] POST /20231130/actions/chat · rejected: RSA-SHA256 signature does not verify
    ✔ PASS provider surfaced the rejection: model error: oci returned HTTP 401 (NotAuthenticated): oci returned HTTP 401: The required information to complete authentication was not provided or was incorrect.

4/4 checks passed
```

## 1-tinyinference-122-test-suite

```text
# tinyinference#122 — full verification suite (fmt, clippy -D warnings, build, test, doc)

BUILD_OK
CLIPPY_OK
DOC_OK
FMT_OK
TEST_OK

Test totals:
  passed 1633 failed 0 ignored 1
  OCI provider tests: 87 passed
```
