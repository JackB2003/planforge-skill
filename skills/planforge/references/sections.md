# The 16 Plan Sections

Every plan artifact is a JSON object `{ "sections": [ ...16 section objects, in this order... ] }`.

Every section follows this exact shape:

```json
{
  "id": "architecture",
  "title": "App Architecture Framework",
  "overview": "2-3 sentence summary of the decisions made for this section",
  "decisions": [{ "choice": "...", "rationale": "..." }],
  "implementation": { "...section-specific fields, see below..." },
  "risks": [{ "risk": "...", "mitigation": "..." }],
  "agent_notes": ["specific instructions for build agents working on this section"]
}
```

**Non-negotiable rules for every section, in every round:**
- Non-empty `overview`, at least one `decisions` entry, populated `implementation` fields, at least one `risks` entry with a mitigation, at least one `agent_notes` entry.
- `implementation` fields must be specific — name actual tools, frameworks, packages, configs. Never write "TBD" or "depends on requirements."
- If content originates from an imported source document, preserve its real decisions and rationale faithfully. Only invent content where the source is genuinely silent, and when you do, prefix every `agent_notes` entry for that section with `[GAP-FILLED]`.
- All 16 sections must be present, in order, every round — even sections nothing changed about get carried forward.

## Section definitions and required `implementation` fields

1. **`architecture`** — App Architecture Framework
   `{ frontend_framework, frontend_language, backend_pattern, backend_framework, api_style, repo_structure, monorepo_tool, state_management, rendering_strategy, key_libraries }`

2. **`database`** — Database Setup
   `{ primary_db, orm_or_client, schema_overview: [{ table, purpose, key_fields }], indexing_strategy, migration_strategy, backup_strategy, connection_pooling }`

3. **`server`** — Server Requirements
   `{ host, os, cpu_ram_estimate, ports: [{ port, service }], nginx_config_notes, pm2_processes, env_vars_needed: [{ key, purpose }], domain_routing }`

4. **`environment`** — Environment Strategy
   `{ environments: ["dev","staging","prod"], env_file_strategy, secrets_management, feature_flag_approach, promotion_process }`

5. **`features`** — Features
   `{ mvp_features: [{ name, description, priority, acceptance_criteria }], future_features: [{ name, description, phase }] }`

6. **`navigation`** — Navigation Structure
   `{ routes: [{ path, component, auth_required, description }], layout_structure, nav_components, deep_link_strategy, 404_handling }`

7. **`authentication`** — Authentication
   `{ provider, session_strategy, token_storage, token_expiry, refresh_strategy, roles: [{ name, permissions }], protected_routes, oauth_providers }`

8. **`integrations`** — 3rd-Party Tools & APIs
   `{ integrations: [{ name, purpose, sdk_or_package, auth_method, rate_limits, fallback_strategy }] }`

9. **`security`** — Security Considerations
   `{ rate_limiting: { tool, limits }, input_validation, cors_policy, csp_headers, sql_injection_prevention, xss_prevention, csrf_protection, dependency_audit_tool, secrets_scanning }`

10. **`qa_testing`** — QA Testing Strategy
    `{ unit_testing: { tool, coverage_target }, integration_testing: { tool, scope }, e2e_testing: { tool, key_flows }, test_data_strategy, ci_test_gates, manual_qa_checklist }`

11. **`error_logging`** — Error Logging & Debugging
    `{ client_error_tool, server_error_tool, log_levels, log_destinations, alert_thresholds, debug_output_strategy, source_maps, correlation_ids }`

12. **`code_health`** — Code Health & Structural Integrity
    `{ during_build_standards: [string], static_analysis_tools: [{ tool, purpose, config }], ci_gates: [{ check, blocking }], post_build_audit: { architectural_hotspot_method, dependency_audit_tool, duplicate_detection_tool, dead_code_tool }, auditor_agent_handoff: string }`
    Owned by a dedicated **Code Auditor** subagent, also referenced in section 16, operating in three phases: during-build enforcement, CI/CD gating, post-build audit.

13. **`deployment`** — Deployment & CI/CD
    `{ ci_tool, cd_strategy, build_steps: [string], deploy_steps: [string], rollback_strategy, health_checks, zero_downtime_approach }`

14. **`budget`** — Cost & Budget Estimate
    `{ infrastructure: [{ item, monthly_cost }], third_party_apis: [{ service, free_tier, paid_tier }], total_monthly_estimate_low, total_monthly_estimate_high, cost_optimization_notes }`

15. **`timeline`** — Timeline & Phasing
    `{ phases: [{ name, duration, deliverables: [string], dependencies: [string] }], total_estimate, critical_path_items }`

16. **`agent_team`** — AI Subagent Team
    `{ agents: [{ role, specialty, model, tools_needed, responsibilities: [string], handoff_protocol }] }`
    Must include a dedicated **"Code Auditor"** agent that owns section 12, gap-filled if the source doesn't define one.
