# Pendo plugins for Claude Code

Pendo analytics for Claude Code: account health, feature adoption, session replays, feedback analysis, data-informed planning, and agent analytics setup.

## Plugins

This marketplace contains two plugins:

| Plugin | Description |
|:-------|:------------|
| `pendo-analytics` | Pendo analytics skills for account health, feature adoption, session replays, feedback analysis, and data-informed planning |
| `setup-agent-analytics` | Detect AI agents in your codebase and instrument them with Pendo agent analytics |

## Quickstart

### Option A: Install from Marketplace (Recommended)

1. **Add the marketplace:**
   Inside Claude Code, run:
   ```
   /plugin marketplace add pendo-io/claude-pendo-plugin
   ```

2. **Install plugins:**
   Run `/plugin`, navigate to the **Marketplaces** section, select the **pendo-plugins** marketplace, and install the plugins you want.

3. **Authenticate the Pendo MCP server:**
   Run the `/mcp` command inside Claude Code and follow the authentication flow.

4. **Run a skill:**

   **pendo-analytics:**
   ```
   /pendo-analytics:account-health <account-name>
   /pendo-analytics:feature-adoption <feature-name>
   /pendo-analytics:feedback-analysis
   /pendo-analytics:session-replay
   /pendo-analytics:data-informed-planning <planning question>
   ```

   **setup-agent-analytics:**
   ```
   /setup-agent-analytics:setup-agent-analytics [agent-id-1] [agent-id-2] ...
   ```

### Option B: Manual Installation

1. **Clone the repo:**
   ```bash
   git clone https://github.com/pendo-io/claude-pendo-plugin.git
   ```

2. **Start Claude Code with the plugin:**
   ```bash
   claude --plugin-dir /path/to/claude-pendo-plugin
   ```

3. **Authenticate the Pendo MCP server:**
   Run the `/mcp` command inside Claude Code and follow the authentication flow.

4. **Run a skill:**

   **pendo-analytics:**
   ```
   /pendo-analytics:account-health <account-name>
   /pendo-analytics:feature-adoption <feature-name>
   /pendo-analytics:feedback-analysis
   /pendo-analytics:session-replay
   /pendo-analytics:data-informed-planning <planning question>
   ```

   **setup-agent-analytics:**
   ```
   /setup-agent-analytics:setup-agent-analytics [agent-id-1] [agent-id-2] ...
   ```

## Components

### pendo-analytics skills

| Skill | Description |
|:------|:------------|
| `account-health` | Prepare for a customer call by synthesizing engagement, sentiment, and feedback from Pendo analytics |
| `feature-adoption` | Analyze feature adoption rates, identify power users vs laggards, and track adoption trends |
| `feedback-analysis` | Deep analysis of customer feedback - discover themes, extract insights, and identify risks |
| `session-replay` | Find and surface relevant session replays for debugging, UX research, and understanding user behavior |
| `data-informed-planning` | Ground a planning task in real product data — gathers signals, issues, wiki, customer feedback, PES/NPS, segments, and session replays from Novus + Pendo, then delegates to `/superpowers:writing-plans` |

### setup-agent-analytics skills

| Skill | Description |
|:------|:------------|
| `setup-agent-analytics` | Detect AI agents in a codebase and instrument them with Pendo `trackAgent()` calls or the server-side Conversations API |

### MCP Servers

The `pendo-analytics` plugin auto-configures two MCP servers:

| Server | URL | Purpose |
|:-------|:----|:--------|
| `pendo-external` | `https://app.pendo.io/mcp/v0/shttp` | Pendo analytics, feedback, session replays, segments |
| `novus` | `https://novus-api.pendo.io/mcp` | Novus artifact graph, signals, issues, product wiki |

Run `/mcp` once after installing the plugin to authenticate each server.

### Pendo MCP tools (`pendo-external`)

| Tool | Purpose |
|:-----|:--------|
| `activityQuery` | Engagement metrics and activity data |
| `productEngagementScore` | PES calculations |
| `searchEntities` | Find accounts, pages, features |
| `accountQuery` | Account metadata |
| `accountMetadataSchema` | Account metadata schema |
| `visitorQuery` | Visitor metadata |
| `sessionReplayList` | Find session recordings |
| `generate_feedback_topics` | Cluster feedback into themes |
| `get_feedback_insights` | Extract key insights |
| `get_feedback_items` | Raw feedback data |
| `get_ideas` | Voice of Customer ideas / feature requests |
| `guideMetrics` | Guide performance metrics |
| `npsScore` | NPS scores (subscription must include NPS) |
| `segmentList` | Available segments |
| `list_all_applications` | List Pendo applications |

### Novus MCP tools (`novus`)

| Tool | Purpose |
|:-----|:--------|
| `get_pendo_apps` | List Novus-tracked apps |
| `get_product_wiki` | Auto-generated product wiki / sitemap |
| `list_signals` | AI-generated product signals (issues, opportunities, trends) |
| `list_issues` | AI-detected issues with recommended actions |
| `list_artifacts` / `get_artifact` | Artifact graph nodes (pages, features, funnels, journeys, etc.) |
| `get_related_artifacts` / `get_artifact_subtree` | Artifact relationships |
| `get_page_metrics` / `get_feature_metrics` | Pre-processed engagement metrics |
| `get_funnel_analysis` / `get_journey_analysis` | Conversion path analysis |
| `get_headline_metrics` | Top-line product health |
| `preview_guide` | Render a Pendo guide preview |
| `create_flag` / `update_flag` | Feature flag tooling |

### Skill Dependencies

`data-informed-planning` delegates plan-writing to `/superpowers:writing-plans`. Install the [superpowers plugin](https://github.com/obra/superpowers) alongside this one to get the full workflow.

## License

MIT
