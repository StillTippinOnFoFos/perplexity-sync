# 🔄 Perplexity GitHub Sync

> **An open-source serverless integration that automatically syncs content from Perplexity AI conversations to GitHub repositories via pull requests.**

## Overview

This project provides a seamless way to sync documentation, notes, and structured content from Perplexity AI directly to your GitHub repositories. Instead of copy-pasting, it creates automated pull requests for review and version control.

## 🎯 Project Goals

- **Automated Sync**: Push content from Perplexity conversations directly to GitHub
- **Version Control**: All changes go through GitHub's PR review process
- **Security**: Private repositories with secure token management
- **Serverless**: Zero-maintenance deployment with free hosting
- **Flexible**: Works with any type of documentation or structured content

## 🏗️ Architecture

```
┌─────────────────┐    HTTP POST    ┌─────────────────┐    GitHub API    ┌─────────────────┐
│                 │ ─────────────> │                 │ ──────────────> │                 │
│   Perplexity    │   (Content)     │ Cloudflare      │  (Branch + PR)   │   GitHub Repo   │
│   Assistant     │                 │ Worker          │                 │                 │
└─────────────────┘                 └─────────────────┘                 └─────────────────┘
```

### Components

1. **Perplexity Integration**: AI assistant generates and manages content
2. **Cloudflare Worker**: Serverless API endpoint that receives updates and creates GitHub PRs
3. **GitHub Repository**: Version-controlled storage for all your content
4. **Pull Request Workflow**: Review and approve changes before they're merged

## 🚀 Quick Start

### Prerequisites

- [Node.js](https://nodejs.org/) (v16+)
- [Cloudflare Account](https://cloudflare.com) (free tier works)
- [GitHub Personal Access Token](https://github.com/settings/tokens) with repo permissions
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/install-and-update/)

### Installation

1. **Clone and setup the project:**
   ```bash
   git clone https://github.com/yourusername/perplexity-github-sync.git
   cd perplexity-github-sync
   npm install
   ```

2. **Configure your worker:**
   ```bash
   # Copy and edit the configuration file
   cp wrangler.toml.example wrangler.toml
   # Edit wrangler.toml with your repository details
   ```

3. **Set up secrets:**
   ```bash
   # Add your GitHub Personal Access Token
   wrangler secret put GITHUB_TOKEN
   
   # Optional: Add API key for additional endpoint security
   wrangler secret put API_KEY
   ```

4. **Deploy the Worker:**
   ```bash
   wrangler deploy
   ```

5. **Note your endpoint URL** (e.g., `https://your-worker.yourname.workers.dev`)

## 🔒 GitHub Security Configuration

### Creating a Secure Personal Access Token

1. **Go to GitHub Settings** → Developer settings → Personal access tokens → Tokens (classic)

2. **Create a new token** with minimal required permissions:
   - ✅ **repo** (Full control of private repositories)
     - repo:status
     - repo_deployment  
     - public_repo (only if you want to sync to public repos)
   - ❌ **Do NOT grant** admin, delete, or user permissions

3. **Scope the token to specific repositories** (recommended):
   - Instead of granting access to all repos, use fine-grained tokens
   - Go to Personal access tokens → Fine-grained tokens
   - Select only the repositories you want to sync to
   - Grant only "Contents" and "Pull requests" permissions

### Repository Security Best Practices

#### Option 1: Dedicated Sync Repository (Recommended)
```bash
# Create a separate private repository just for Perplexity syncing
gh repo create my-perplexity-docs --private
# Use this repo in your REPO environment variable
```

Benefits:
- Isolates sync content from your main repositories
- Easier to manage permissions and access
- Can be shared selectively with collaborators

#### Option 2: Branch Protection for Existing Repository
If syncing to an existing repository:

1. **Enable branch protection rules**:
   - Go to Settings → Branches
   - Add rule for your default branch (main/master)
   - Enable "Require pull request reviews before merging"
   - Enable "Restrict pushes that create files"

2. **Create a dedicated folder**:
   ```
   your-repo/
   ├── perplexity-sync/
   │   ├── notes/
   │   ├── documentation/
   │   └── README.md
   └── (your other files)
   ```

### Worker Security Configuration

#### Environment Variables Security
```toml
# In wrangler.toml - never put secrets here!
[vars]
REPO = "yourusername/your-private-repo"
BASE_BRANCH = "main"

# Set secrets via CLI only:
# wrangler secret put GITHUB_TOKEN
# wrangler secret put API_KEY
```

#### Optional API Key Protection
Add an additional layer of security by requiring an API key:

1. **Generate a random API key:**
   ```bash
   # Generate a secure random key
   openssl rand -hex 32
   ```

2. **Set it as a secret:**
   ```bash
   wrangler secret put API_KEY
   ```

3. **Include in requests from Perplexity:**
   ```
   Headers: Authorization: Bearer your-api-key-here
   ```

### Access Logging and Monitoring

1. **Enable Cloudflare Analytics** (free tier includes basic analytics)

2. **Monitor your GitHub token usage**:
   - Go to GitHub Settings → Developer settings → Personal access tokens
   - Check "Last used" dates to ensure your token isn't being misused

3. **Set up repository notifications**:
   - Go to your repository → Settings → Notifications
   - Enable email notifications for pull requests

## 📖 Usage

### Basic Sync from Perplexity

1. **In your Perplexity conversation:**
   ```
   Please sync this content to my GitHub repository as "notes/project-ideas.md":

   # Project Ideas
   - Build a documentation system
   - Create automated workflows
   - Implement version control for notes
   ```

2. **Perplexity posts to your endpoint** with the filename and content

3. **Review the pull request** on GitHub and merge when ready

### API Endpoint

**POST** `https://your-worker.yourname.workers.dev`

**Headers:**
```
Content-Type: application/json
Authorization: Bearer your-api-key (optional)
```

**Body:**
```json
{
  "filename": "path/to/file.md",
  "content": "# Your content here\n\nMarkdown or any text content",
  "commit_message": "Optional custom commit message"
}
```

**Response:**
```json
{
  "success": true,
  "pr_url": "https://github.com/user/repo/pull/123",
  "pr_number": 123,
  "branch": "perplexity-sync-2025-10-07T22-13-45-123Z"
}
```

## 📁 Project Structure

```
perplexity-github-sync/
├── src/
│   └── index.js              # Main Cloudflare Worker code
├── docs/
│   ├── setup-guide.md        # Detailed setup instructions
│   ├── security-guide.md     # Security best practices
│   └── api-reference.md      # Complete API documentation
├── examples/
│   ├── basic-usage/          # Simple examples
│   └── advanced-workflows/   # Complex integration examples
├── wrangler.toml.example     # Cloudflare Worker configuration template
├── package.json
├── LICENSE
└── README.md
```

## 🔧 Configuration Options

### Cloudflare Worker Settings

Edit `wrangler.toml`:
```toml
name = "your-sync-worker"
main = "src/index.js"
compatibility_date = "2023-10-30"

[vars]
REPO = "yourusername/your-repo"
BASE_BRANCH = "main"

# Optional: Customize PR behavior
PR_TITLE_PREFIX = "📝 Perplexity Sync: "
AUTO_MERGE = "false"
```

### GitHub Repository Structure

Recommended organization for synced content:
```
your-repo/
├── perplexity-sync/
│   ├── notes/
│   │   ├── daily-notes.md
│   │   └── project-ideas.md
│   ├── documentation/
│   │   ├── api-docs.md
│   │   └── user-guides.md
│   ├── research/
│   │   └── market-analysis.md
│   └── README.md
├── .github/
│   └── workflows/
│       └── validate-sync.yml  # Optional: validate synced content
└── README.md
```

## 🔒 Security Features

- **Private repository support** with secure token handling
- **Pull request workflow** prevents direct commits to main branch
- **API key authentication** for additional endpoint security
- **Minimal GitHub permissions** required
- **Branch isolation** keeps each sync in a separate branch
- **Audit trail** through GitHub's built-in change tracking

## 💡 Use Cases

- **Research Notes**: Sync research findings and analysis
- **Documentation**: Maintain living documentation with AI assistance
- **Meeting Notes**: Structured meeting notes with action items
- **Project Planning**: Project roadmaps and planning documents
- **Knowledge Base**: Build searchable knowledge repositories
- **Code Documentation**: API docs and technical specifications

## 🤝 Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Areas for Contribution

- Enhanced error handling and retry logic
- Support for multiple repository destinations
- Integration with other documentation platforms
- Advanced content transformation and formatting
- Webhook integrations for automated workflows

## 📄 License

MIT License - see [LICENSE](LICENSE) for details.

## 🙏 Acknowledgments

- **Generated by**: [Perplexity AI](https://perplexity.ai) - Initial codebase and documentation
- **Reviewed and maintained by**: [@yourusername](https://github.com/yourusername) - Human oversight and quality assurance
- Built for seamless AI-human collaboration workflows
- Powered by [Cloudflare Workers](https://workers.cloudflare.com)
- Inspired by the need for better AI-assisted documentation workflows

## 📞 Support

- **Documentation**: Check the `/docs/` folder for detailed guides
- **Issues**: Report bugs or request features via GitHub Issues
- **Discussions**: Join the conversation in GitHub Discussions
- **Security**: Report security issues privately via email

---

*Streamline your AI-assisted workflows with version-controlled collaboration! 🤖*
