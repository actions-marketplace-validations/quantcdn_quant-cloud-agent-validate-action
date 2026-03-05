# Quant Agent Validate Action

A GitHub Action for validating AI agent output against a quality spec using the Quant validation API. Runs behavioral, constraint, and quality checks to produce a scored quality report.

## Features

- **Quality Scoring**: Weighted scoring across behavioral, constraint, and quality check categories
- **Threshold Enforcement**: Configurable pass/fail threshold defined in spec
- **Detailed Reports**: Returns full quality report with per-check results and suggestions
- **CI Integration**: Automatically fails the workflow step if validation does not pass
- **Flexible Spec Input**: Load spec from file path or pass as inline JSON

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `quant_api_key` | Yes | Quant API key |
| `quant_organization` | Yes | Quant organisation name |
| `agent_id` | Yes | Agent UUID to validate |
| `agent_output` | Yes | JSON string of the agent tool call output |
| `validation_spec` | Yes | Path to validation spec JSON file, or inline JSON string |
| `conversation_context` | No | JSON string of conversation context (tool calls, message count) |
| `test_fixtures` | No | Path to JSON file with test fixture state |
| `git_sha` | No | Git SHA that triggered this validation (auto-detected if not set) |
| `base_url` | No | Quant API base URL |

## Outputs

| Output | Description |
|--------|-------------|
| `score` | Quality score (0.0-1.0) |
| `passed` | Whether validation passed (`true`/`false`) |
| `report` | Full QualityReport JSON |
| `summary` | Human-readable summary string |

## Validation Spec Format

```json
{
  "name": "assessment-quality",
  "threshold": 0.7,
  "checks": {
    "behavioral": [
      {
        "id": "calls-submit",
        "description": "Agent calls submit_assessment tool",
        "weight": 3,
        "assertion": {
          "type": "tool_called",
          "tool": "submit_assessment"
        }
      }
    ],
    "constraint": [
      {
        "id": "scores-in-range",
        "description": "All scores are between 1 and 10",
        "weight": 2,
        "assertion": {
          "type": "field_range",
          "path": "items[*].score",
          "min": 1,
          "max": 10
        }
      }
    ],
    "quality": [
      {
        "id": "reasoning-length",
        "description": "Each reasoning field is at least 50 chars",
        "weight": 2,
        "assertion": {
          "type": "field_length",
          "path": "items[*].reasoning",
          "min": 50
        }
      }
    ]
  }
}
```

### Available Assertion Types

| Type | Purpose | Key fields |
|------|---------|------------|
| `tool_called` | Agent called a specific tool | `tool` |
| `tool_called_with` | Tool called with specific param value | `tool`, `path`, `expected` |
| `field_length` | String field meets min/max length | `path`, `min`, `max` |
| `field_range` | Numeric field within range | `path`, `min`, `max` |
| `field_in_enum` | Field value is in allowed set | `path`, `values` |
| `array_ratio` | Array has min ratio of items matching | `path`, `condition`, `min_ratio` |
| `distinct_values` | Array has enough unique values | `path`, `min` |
| `output_contains_when` | Conditional content check | `condition`, `path`, `contains` |

## Usage

### Basic Usage

```yaml
- name: Validate Agent Output
  uses: quantcdn/quant-cloud-agent-validate-action@v1
  with:
    quant_api_key: ${{ secrets.QUANT_API_KEY }}
    quant_organization: ${{ secrets.QUANT_ORGANIZATION }}
    agent_id: ${{ vars.AGENT_ID }}
    agent_output: ${{ steps.dispatch.outputs.agent_output }}
    validation_spec: agents/my-agent/validation-spec.json
```

### With Conversation Context and Fixtures

```yaml
- name: Validate Agent Output
  uses: quantcdn/quant-cloud-agent-validate-action@v1
  with:
    quant_api_key: ${{ secrets.QUANT_API_KEY }}
    quant_organization: ${{ secrets.QUANT_ORGANIZATION }}
    agent_id: ${{ vars.AGENT_ID }}
    agent_output: ${{ steps.dispatch.outputs.agent_output }}
    validation_spec: agents/my-agent/validation-spec.json
    conversation_context: ${{ steps.dispatch.outputs.conversation_context }}
    test_fixtures: agents/my-agent/fixtures.json
```

### Complete Pipeline Example

```yaml
name: Agent Validation Pipeline
on:
  push:
    branches: [main]
    paths: ['agents/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy Agent
        uses: quantcdn/quant-cloud-agent-deploy-action@v1
        with:
          quant_api_key: ${{ secrets.QUANT_API_KEY }}
          quant_organization: ${{ secrets.QUANT_ORGANIZATION }}
          agents_dir: agents/

      - name: Dispatch Agent
        uses: quantcdn/quant-cloud-agent-dispatch-action@v1
        id: dispatch
        with:
          quant_api_key: ${{ secrets.QUANT_API_KEY }}
          quant_organization: ${{ secrets.QUANT_ORGANIZATION }}
          agent_id: ${{ vars.AGENT_ID }}
          test_fixtures: agents/my-agent/fixtures.json

      - name: Validate Agent Output
        uses: quantcdn/quant-cloud-agent-validate-action@v1
        id: validate
        with:
          quant_api_key: ${{ secrets.QUANT_API_KEY }}
          quant_organization: ${{ secrets.QUANT_ORGANIZATION }}
          agent_id: ${{ vars.AGENT_ID }}
          agent_output: ${{ steps.dispatch.outputs.agent_output }}
          validation_spec: agents/my-agent/validation-spec.json
          conversation_context: ${{ steps.dispatch.outputs.conversation_context }}
          test_fixtures: agents/my-agent/fixtures.json

      - name: Validation Summary
        if: always()
        run: |
          echo "Score: ${{ steps.validate.outputs.score }}"
          echo "Passed: ${{ steps.validate.outputs.passed }}"
          echo "Summary: ${{ steps.validate.outputs.summary }}"
```

## Error Handling

The action will fail if:
- The API key or organization is invalid
- The validation spec file is not found or is invalid JSON
- The spec is missing required `name` or `checks` fields
- The agent output is not valid JSON
- The validation score falls below the threshold
- The API returns an unexpected error

## Development

### Building

```bash
npm install
npm run build
```

### Testing

```bash
npm test
```

## License

This project is licensed under the MIT License.
