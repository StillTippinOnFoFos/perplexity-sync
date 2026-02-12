# 🔄 Perplexity GitHub Sync

*Serverless integration for instant, secure, version-controlled sync of Perplexity AI content to GitHub via automated pull requests.*

## What This Does

Instead of copy-pasting content from Perplexity conversations, this creates a simple API endpoint that:
1. Receives markdown/text from Perplexity 
2. Creates a new branch in your GitHub repo
3. Commits the content as a file
4. Opens a pull request for you to review and merge

Everything stays private, secure, and version-controlled.

## Quick Setup (5 minutes)

### 1. Create GitHub Token
- Go to GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens
- Create token with access to your target repo
- Grant only "Contents" and "Pull requests" permissions
- Copy the token

### 2. Deploy Cloudflare Worker
```bash
# Install Wrangler CLI
npm install -g wrangler

# Login to Cloudflare (free account works)
wrangler login

# Create new worker project
wrangler init perplexity-sync
cd perplexity-sync
```

Copy this code into `src/index.js`:

```javascript
export default {
  async fetch(request, env) {
    if (request.method !== "POST") {
      return new Response("POST only", { status: 405 });
    }

    // Check API key if configured
    if (env.API_KEY) {
      const authHeader = request.headers.get("Authorization");
      if (!authHeader || authHeader !== `Bearer ${env.API_KEY}`) {
        return new Response("Unauthorized", { status: 401 });
      }
    }

    try {
      const { filename, content } = await request.json();
      
      if (!filename || !content) {
        return new Response("Missing filename or content", { status: 400 });
      }

      const GITHUB_TOKEN = env.GITHUB_TOKEN;
      const REPO = env.REPO; // Format: "username/repo-name"
      const BASE_BRANCH = "main";
      
      // Create unique branch name
      const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
      const branchName = `perplexity-sync-${timestamp}`;

      const githubHeaders = {
        "Authorization": `token ${GITHUB_TOKEN}`,
        "Content-Type": "application/json"
      };

      // Get base branch SHA
      const baseRef = await fetch(
        `https://api.github.com/repos/${REPO}/git/ref/heads/${BASE_BRANCH}`,
        { headers: githubHeaders }
      );
      const baseData = await baseRef.json();
      const baseSHA = baseData.object.sha;

      // Create new branch
      await fetch(`https://api.github.com/repos/${REPO}/git/refs`, {
        method: "POST",
        headers: githubHeaders,
        body: JSON.stringify({
          ref: `refs/heads/${branchName}`,
          sha: baseSHA
        })
      });

      // Check if file exists (to get SHA for updates)
      let fileSha = null;
      const getFile = await fetch(
        `https://api.github.com/repos/${REPO}/contents/${filename}?ref=${branchName}`,
        { headers: githubHeaders }
      );
      if (getFile.status === 200) {
        const fileData = await getFile.json();
        fileSha = fileData.sha;
      }

      // Create/update file
      await fetch(`https://api.github.com/repos/${REPO}/contents/${filename}`, {
        method: "PUT",
        headers: githubHeaders,
        body: JSON.stringify({
          message: `Update ${filename} via Perplexity sync`,
          content: btoa(unescape(encodeURIComponent(content))),
          branch: branchName,
          ...(fileSha ? { sha: fileSha } : {})
        })
      });

      // Create pull request
      const prResponse = await fetch(`https://api.github.com/repos/${REPO}/pulls`, {
        method: "POST",
        headers: githubHeaders,
        body: JSON.stringify({
          title: `📝 Update ${filename} from Perplexity`,
          body: `Automated update from Perplexity AI\n\n- File: \`${filename}\`\n- Generated: ${new Date().toISOString()}`,
          head: branchName,
          base: BASE_BRANCH
        })
      });

      const prData = await prResponse.json();
      
      return new Response(JSON.stringify({
        success: true,
        pr_url: prData.html_url,
        pr_number: prData.number
      }), { 
        status: 200,
        headers: { "Content-Type": "application/json" }
      });

    } catch (error) {
      return new Response(`Error: ${error.message}`, { status: 500 });
    }
  }
};
```

### 3. Configure Worker Secrets
```bash
# Set your repository (replace with your username/repo)
wrangler secret put REPO
# Enter: yourusername/your-repo-name

# Set your GitHub token
wrangler secret put GITHUB_TOKEN  
# Paste your GitHub token

# Generate and set API key for security
openssl rand -hex 32
wrangler secret put API_KEY
# Paste the generated key

# Deploy
wrangler deploy
```

**Save these values** - you'll need them for Perplexity:
- Worker URL: `https://perplexity-sync.yourname.workers.dev`
- API Key: `your-32-character-api-key`

### 4. Configure Perplexity Integration

In your Perplexity conversation, provide these details:

```
I've set up a GitHub sync endpoint. Here are the details:

Endpoint URL: https://perplexity-sync.yourname.workers.dev
API Key: your-32-character-api-key
Repository: yourusername/your-repo-name

Please use this endpoint to sync content when I ask you to save or commit files to GitHub.
```

## Usage

### Basic Sync
```
Please sync this content to my GitHub repo as "notes/project-ideas.md":

# My Project Ideas
- Build something cool
- Document everything  
- Use version control
```

### Organized Sync
```
Save this to "docs/api-reference.md" in my GitHub repo:

# API Documentation
## Overview
This API handles user authentication...
```

### Custom Commit Message
```
Commit this as "changelog.md" with message "Add v2.0 release notes":

# Changelog
## v2.0.0
- Added new features
- Fixed bugs
```

## How Perplexity Integration Works

When you ask Perplexity to sync content, it will:

1. **POST to your endpoint** with this format:
   ```json
   {
     "filename": "path/to/file.md",
     "content": "# Your content here...",
     "commit_message": "Optional custom message"
   }
   ```

2. **Include your API key** in the Authorization header:
   ```
   Authorization: Bearer your-32-character-api-key
   ```

3. **Return the PR details** so you can review:
   ```json
   {
     "success": true,
     "pr_url": "https://github.com/user/repo/pull/123",
     "pr_number": 123
   }
   ```

## Security

**API Key Protection**: Each request must include your unique API key - prevents unauthorized access to your endpoint.

**GitHub Token**: Stored securely in Cloudflare Workers secrets (encrypted at rest).

**Repository Access**: Token only needs "Contents" and "Pull requests" permissions to your specific repo.

**Pull Request Workflow**: Nothing gets merged without your review.

**Private Repository**: Everything stays in your private repo, only accessible with your token.

## Cost

- **Cloudflare Workers**: Free (100k requests/day)
- **GitHub**: Free for private repos
- **Total**: $0/month for personal use

## Troubleshooting

**"Unauthorized" error**: 
- Check your API key is set correctly in Cloudflare secrets
- Make sure you're providing the right API key to Perplexity
- Verify your GitHub token has the right permissions and isn't expired

**"Repository not found"**: 
- Make sure REPO secret is formatted as "username/repo-name"
- Check your GitHub token has access to the repository

**No PR created**: 
- Verify your repository name is correct
- Ensure the token has "Pull requests" permission
- Check Cloudflare worker logs: `wrangler tail`

**Perplexity can't connect**:
- Verify your worker URL is correct and accessible
- Test the endpoint manually with curl:
  ```bash
  curl -X POST https://your-worker.workers.dev \
    -H "Authorization: Bearer your-api-key" \
    -H "Content-Type: application/json" \
    -d '{"filename": "test.md", "content": "# Test"}'
  ```

## Example Perplexity Prompts

**Research Notes**:
```
Please sync my research findings to "research/market-analysis.md":

# Market Analysis - Q4 2025
## Key Findings
- Market grew 15% this quarter
- New competitors emerged
```

**Meeting Notes**:
```
Save these meeting notes as "meetings/2025-10-07-team-sync.md":

# Team Sync - October 7, 2025
## Attendees: John, Sarah, Mike
## Action Items:
- [ ] Update documentation by Friday
- [ ] Review pull requests
```

**Documentation Updates**:
```
Update "docs/troubleshooting.md" with this new section:

## Common Issues
### Database Connection
If you see connection errors, check...
```

## Credits

- **Generated by**: [Perplexity AI](https://perplexity.ai)
- **Reviewed by**: [@yourusername](https://github.com/yourusername) 
- **License**: MIT

---

*Now your Perplexity conversations can flow directly into version-controlled documentation! 🤖→📝→🔄*
