# Pendo plugins for Claude Code

Pendo analytics for Claude Code: account health, feature adoption, session replays, feedback analysis, and agent analytics setup.

## Plugins

This marketplace contains two plugins:

| Plugin | Description |
|:-------|:------------|
| `pendo-analytics` | Pendo analytics skills for account health, feature adoption, session replays, and feedback analysis |
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

### setup-agent-analytics skills

| Skill | Description |
|:------|:------------|
| `setup-agent-analytics` | Detect AI agents in a codebase and instrument them with Pendo `trackAgent()` calls or the server-side Conversations API |

### MCP Tools

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
| `guideMetrics` | Guide performance metrics |
| `segmentList` | Available segments |
| `list_all_applications` | List Pendo applications |

## License

MIT
