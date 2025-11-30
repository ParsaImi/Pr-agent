# PR Agent

<<<<<<< HEAD
## Setup shared secret token using in both gitlab and pr agent
SHARED_SECRET=$(python -c "import secrets; print(secrets.token_hex(10))")
=======
## GitHub Public Repository Setup

Add this to your GitHub Actions workflow:

```yaml
name: PR Agent (Gemini)
on:
  pull_request:
    types: [opened, reopened, ready_for_review]
  issue_comment:
jobs:
  pr_agent_job:
    if: ${{ github.event.sender.type != 'Bot' }}
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
      contents: write
    steps:
      - name: PR Agent action step
        uses: qodo-ai/pr-agent@main
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          config.model: "gemini/gemini-2.5-flash"
          config.fallback_models: '["gemini-2.5-flash"]'
          GOOGLE_AI_STUDIO.GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
          github_action_config.auto_review: "true"
          github_action_config.auto_describe: "true"
          github_action_config.auto_improve: "true"
```

Add `GEMINI_API_KEY` to your GitHub secrets.

In my case, I used the Gemini 2.5-flash model.

Code Review, Code Improvement, and Code Description will be automatically triggered when a PR is opened or reopened.

---

## GitLab Repository Setup

### GitLab User for Code Review and Code Improvement

Create a bot user for PR Agent and give it the reporter role (or just use the default `@git` user).
>>>>>>> c5e3564 (update readme)

### Configure PR Agent to Interact with GitLab

PR Agent persists configuration in this order: `.env` > `.pr-agent.toml` > `pr-agent/settings/configuration.toml`

Make the changes you want in `.env` (in the root directory where you deploy PR Agent):

```env
gitlab__url=YOUR_GITLAB_URL
gitlab__personal_access_token=YOUR_PERSONAL_ACCESS_TOKEN
gitlab__shared_secret=YOUR_SHARED_SECRET
config__git_provider=gitlab
config__model=huggingface/openai/gpt-oss-120b:fastest
config__fallback_models='["huggingface/together/deepseek-ai/DeepSeek-R1"]'
HUGGINGFACE__key=YOUR HF API KEY
HUGGINGFACE__API_BASE=https://router.huggingface.co/v1
config__custom_model_max_tokens=200000
```

I will explain each of the configurations in detail.

#### Configuration Parameters

**`gitlab__url`** - Main URL of your GitLab instance.

**`gitlab__personal_access_token`** - Personal access token of your bot user. This token will be used to verify that PR Agent can use that user (bot).

#### Setup Shared Secret Token

This shared token will be used to verify the authenticity of the webhook request.

To generate a shared secret token:

```bash
python -c "import secrets; print(secrets.token_hex(10))"
```

Copy the result value and use it in the `gitlab__shared_secret` variable.

#### Git Provider

**`config__git_provider`** - The git provider that PR Agent will use to interact with your repository.

#### AI Model for Code Review and Code Improvement

I have used different models from OpenAI, Gemini, DeepSeek, and Cohere, but some of them are not available for Iranian users and some of them have limitations for using their API.

However, Hugging Face worked for me, and I have used `huggingface/openai/gpt-oss-120b:fastest` as the main model and `huggingface/together/deepseek-ai/DeepSeek-R1` as the fallback model.

#### Use Hugging Face API

Copy your API key from the Hugging Face website in Settings > Access Tokens and use it in the `HUGGINGFACE__key` variable.

#### Use Hugging Face API Base

The `HUGGINGFACE__API_BASE` configuration parameter specifies the base URL endpoint for making API requests to Hugging Face's services.

#### Maximum Context Window Size (in Tokens)

This tells the application:

- How much text (measured in tokens) can be included in a single request to the model
- It's the total "memory" the model can work with in one interaction

---

## More Configurations in `pr_agent.toml`

Sets the maximum time (in seconds) the AI can take to respond to a request:

```toml
ai_timeout = 300
```

Specifies the language for all AI-generated responses:

```toml
response_language = "en-US"
```

Skip processing PRs whose titles match these regex patterns (useful for automated PRs like dependency updates or bot-generated PRs that don't need review):

```toml
ignore_pr_title = ["^\\[Auto\\]", "^Auto"]
```

Skip processing PRs that have the label "invalid" (allows you to manually exclude PRs from AI review by adding this label):

```toml
ignore_pr_labels = ['invalid']
```

Exclude files matching this glob pattern from AI review:

- `dist/**` - Ignores everything in the dist folder (typically build artifacts)
- Prevents wasting tokens reviewing generated/compiled files

```toml
[ignore]
glob = ['dist/**']
```

### PR Reviewer Configuration

Configures the PR review behavior with custom instructions that shape how the AI reviews code:

```toml
[pr_reviewer]
extra_instructions = """\
(1) Act as a highly experienced software engineer
(2) Provide thorough review of code, documents, and articles
(3) Suggest concrete code snippets for improvement

(4) **Never** comment on indentation, whitespace, blank lines, or other purely stylistic issues unless they change program semantics.

(5) **Priority Review Areas - Check systematically:**
- **Security**: Plaintext passwords, SQL injection, input validation, exception leaks
- **Error Handling**: Bare except clauses, missing try-catch, silent error suppression  
- **Resource Management**: Missing context managers (with statements), unclosed connections/files
- **Type Safety**: Missing type hints, incorrect type usage, unjustified Any types
- **Performance**: Inefficient algorithms (O(n²) or worse), unnecessary loops, memory leaks
- **Code Quality**: Magic numbers, unclear variable names, unused imports/variables
- **API Design**: Missing input validation, no error responses, required field checks
- **Architecture**: Single responsibility violations, tight coupling, global state usage

(6) Focus on concrete, actionable issues with specific code examples and fix recommendations.
"""
num_code_suggestions = 5
inline_code_comments = true
ask_and_reflect = true
```

**`num_code_suggestions = 5`** - Generate up to 5 code improvement suggestions per review.

**`inline_code_comments = true`** - Post review comments directly on specific code lines in the PR. Makes feedback contextual and easier to address.

**`ask_and_reflect = true`** - Enables interactive mode where the AI can ask clarifying questions. Allows the AI to reflect on its suggestions before finalizing.

### PR Description Configuration

Configures automatic PR description generation:

```toml
[pr_description]
extra_instructions = "Generate clear and comprehensive descriptions"
generate_ai_title = true
add_original_user_description = false 
publish_description_as_comment = true
```

**`generate_ai_title`** - Auto-generate a PR title based on the changes.

**`add_original_user_description`** - Don't include the user's original PR description in the AI-generated one.

**`publish_description_as_comment`** - Post the AI-generated description as a comment on the PR (if false, it would overwrite the PR description directly).

### PR Code Suggestions Configuration

Ensures suggestions include concrete code examples, not just descriptions:

```toml
[pr_code_suggestions]
extra_instructions = "Provide actionable code suggestions with examples"
commitable_code_suggestions = true
demand_code_suggestions_self_review = true
```

**`commitable_code_suggestions`** - Generates suggestions as committable patches.

**`demand_code_suggestions_self_review`** - Allows reviewers to apply suggestions directly with a single click.

---

## Configure Webhook

In the repository where you want to use PR Agent, go to Settings > Webhooks > Add Webhook

**URL** - Should have the `/webhook` route, for example: `https://yourpragentserver/webhook`

**Secret Token** - Put the token that you have generated with Python here

**Triggers** should be:

- Push events (All branches)
- Comments
- Merge request events
