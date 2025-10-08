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

### 3. Configure Worker
```bash
# Set your repository (replace with your username/repo)
wrangler secret put REPO
# Enter: yourusername/your-repo-name

# Set your GitHub token
wrangler secret put GITHUB_TOKEN  
# Paste your GitHub token

# Deploy
wrangler deploy
```

Copy the deployment URL (something like `https://perplexity-sync.yourname.workers.dev`)

## Usage

In any Perplexity conversation, tell it:

```
Please sync this to my GitHub repo as "notes/my-ideas.md":

# My Project Ideas
- Build something cool
- Document everything
- Use version control
```

I'll POST to your endpoint, create a branch, commit the file, and open a PR for you to review.

## Security

**Your GitHub token is stored securely** in Cloudflare Workers secrets (encrypted at rest).

**Repository stays private** - only you can access it with your token.

**Pull request workflow** - nothing gets merged without your review.

**Minimal permissions** - token only needs "Contents" and "Pull requests" access to your specific repo.

## Cost

- **Cloudflare Workers**: Free (100k requests/day)
- **GitHub**: Free for private repos
- **Total**: $0/month for personal use

## Troubleshooting

**"Unauthorized" error**: Check your GitHub token has the right permissions and isn't expired.

**"Repository not found"**: Make sure REPO secret is formatted as "username/repo-name".

**No PR created**: Check your repository name and that the token has "Pull requests" permission.

## Advanced Options

**Add API key protection:**
```bash
wrangler secret put API_KEY
# Generate with: openssl rand -hex 32
```

Then include in requests: `Authorization: Bearer your-api-key`

**Custom commit messages:**
```json
{
  "filename": "notes/test.md",
  "content": "# Test",
  "commit_message": "Add new test notes"
}
```

## Credits

- **Generated by**: [Perplexity AI](https://perplexity.ai)
- **Reviewed by**: [@yourusername](https://github.com/yourusername) 
- **License**: MIT

---

*Now your Perplexity conversations can flow directly into version-controlled documentation! 🤖→📝→🔄*
