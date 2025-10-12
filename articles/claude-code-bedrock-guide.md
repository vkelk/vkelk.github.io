# My Journey: Configuring Claude Code Extension with AWS Bedrock (And How You Can Avoid My Mistakes)

## Introduction: Why I Needed This Setup

Like many developers working in enterprise environments, I found myself in a situation where I wanted to leverage AI-assisted coding through Claude Code in VS Code, but I needed to use AWS Bedrock instead of Anthropic's direct API. The reasons were straightforward: my organization already had AWS infrastructure in place, and using Bedrock meant better compliance with our security policies, centralized billing, and integration with our existing AWS services.

What I thought would be a simple configuration turned into several hours of troubleshooting. Status messages like "thinking...", "deliberating...", and "coalescing..." would appear, but no actual responses came through. Error messages about "e is not iterable" filled my developer console, and I couldn't figure out what was wrong.

This guide is the documentation I wish I had when I started. It's based on my real experience, including all the mistakes I made and how I eventually got everything working.

## What You'll Accomplish

By the end of this guide, you'll have:
- Claude Code extension fully integrated with AWS Bedrock
- A working configuration that uses AWS inference profiles
- The ability to use Claude Sonnet 4.5 directly in VS Code through your AWS account
- A troubleshooting framework for common issues

## Prerequisites

Before you begin, ensure you have:

- **Visual Studio Code** installed ([Download here](https://code.visualstudio.com/))
- **AWS CLI** installed and configured ([Installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
- An **AWS account** with appropriate permissions (at minimum: `bedrock:InvokeModel`, `bedrock:ListFoundationModels`, `bedrock:ListInferenceProfiles`)
- Access to **AWS Bedrock** in your chosen region
- The **Claude Code extension** installed in VS Code ([Extension page](https://marketplace.visualstudio.com/items?itemName=Anthropic.claude-code))

## Quick Start Summary - TL;DR

If you're in a hurry, here's the absolute minimum to get started:

1. **Enable Claude in AWS Bedrock** (5 minutes)
   - Console → Bedrock → Model access → Enable Claude Sonnet 4.5

2. **Get your inference profile ARN** (2 minutes)
   ```bash
   aws bedrock list-inference-profiles --region eu-west-2 --profile YOUR_AWS_PROFILE_NAME
   ```

3. **Test AWS connection** (5 minutes)
   ```bash
   echo '{"anthropic_version":"bedrock-2023-05-31","max_tokens":100,"messages":[{"role":"user","content":"Hello"}]}' > request.json
   
   aws bedrock-runtime invoke-model \
     --model-id YOUR_INFERENCE_PROFILE_ARN \
     --body file://request.json \
     --region eu-west-2 \
     --profile YOUR_AWS_PROFILE_NAME \
     --cli-binary-format raw-in-base64-out \
     output.txt
   ```

4. **Configure VS Code** (3 minutes)
   ```json
   {
       "claude-code.selectedModel": "claude-sonnet-4-5-20250929",
       "claude-code.environmentVariables": [
           {"name": "AWS_PROFILE", "value": "YOUR_AWS_PROFILE_NAME"},
           {"name": "AWS_REGION", "value": "eu-west-2"},
           {"name": "BEDROCK_MODEL_ID", "value": "YOUR_INFERENCE_PROFILE_ARN"},
           {"name": "CLAUDE_CODE_USE_BEDROCK", "value": "1"}
       ]
   }
   ```

5. **Reload VS Code and test** (1 minute)
   - Cmd/Ctrl+Shift+P → "Developer: Reload Window"
   - Open Claude Code → Type "say hello"

Total time: ~15 minutes (assuming AWS account already set up)

## Part 1: Understanding and Setting Up AWS Bedrock

### Why AWS Bedrock Instead of Direct API?

AWS Bedrock acts as a unified interface for foundation models, including Claude. For enterprise users, this provides:
- **Compliance**: Keep all AI interactions within your AWS environment
- **Cost Management**: Centralized billing and monitoring through AWS
- **Security**: Leverage existing AWS IAM policies and VPC configurations
- **Integration**: Connect with other AWS services (CloudWatch, Lambda, etc.)


### Step 1: Enable Model Access in AWS Bedrock

**What we're doing:** AWS Bedrock requires explicit permission to access each foundation model. This is a one-time setup per region.

**Why this matters:** Without this step, you'll get authentication errors even if your AWS credentials are valid.

1. Log in to the [AWS Console](https://console.aws.amazon.com/)
2. Navigate to **AWS Bedrock** service (search for "Bedrock" in the services search)
3. **Important:** Select your desired region from the top-right dropdown (e.g., `eu-west-2` for London, `us-east-1` for North Virginia)
4. In the left sidebar, click **Model access**
5. Click **Manage model access** or **Enable specific models**
6. Scroll down to find **Anthropic** section
7. Check the box next to **Claude Sonnet 4.5** (or the specific Claude model you want to use)
8. Click **Request model access** at the bottom
9. Wait for the status to change from "Pending" to "Access granted" (usually instant for Anthropic models)

**Expected outcome:** You should see a green checkmark and "Access granted" status next to Claude Sonnet 4.5.

**Reference:** [AWS Bedrock Model Access Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)

### Step 2: Understanding Inference Profiles (This Was My First Stumbling Block)

**My initial mistake:** I tried using the direct model ID `anthropic.claude-sonnet-4-5-20250929-v1:0` and kept getting errors about on-demand throughput not being supported.

**What I learned:** AWS Bedrock requires **inference profiles** for on-demand usage. An inference profile is essentially a wrapper around the model that provides additional features like cross-region routing, load balancing, and throughput management.

**Why inference profiles exist:**
- They enable cross-region inference (if one region is busy, automatically route to another)
- They provide better availability and reliability
- They allow AWS to manage capacity more efficiently

**Two types of inference profiles:**

**Option A: Pre-configured Cross-Region Profiles (Recommended)**

AWS provides these out of the box:
- **EU regions:** `eu.anthropic.claude-sonnet-4-5-v2:0`
- **US regions:** `us.anthropic.claude-sonnet-4-5-v2:0`

These automatically route requests across multiple regions for better availability.

**Option B: Custom Application Inference Profiles**

You can create custom profiles with specific configurations, but for most use cases, the pre-configured ones work perfectly.

**Reference:** [AWS Bedrock Inference Profiles Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)

### Step 3: Finding Your Inference Profile ARN

**What we're doing:** We need to get the exact Amazon Resource Name (ARN) for our inference profile. This ARN is what Claude Code will use to connect to Bedrock.

**Why the full ARN:** The ARN includes your account ID and region, ensuring requests go to the right place with the right permissions.

Run this command (replace `YOUR_AWS_PROFILE_NAME` with your AWS CLI profile name):

```bash
aws bedrock list-inference-profiles \
  --region eu-west-2 \
  --profile YOUR_AWS_PROFILE_NAME
```

**Expected output:** You'll see a JSON response with available inference profiles. Look for something like:

```json
{
  "inferenceProfileSummaries": [
    {
      "inferenceProfileName": "EU Anthropic Claude Sonnet 4.5",
      "inferenceProfileArn": "arn:aws:bedrock:eu-west-2:ACCOUNT_ID:inference-profile/eu.anthropic.claude-sonnet-4-5-20250929-v1:0",
      "modelArn": "arn:aws:bedrock:eu-west-2::foundation-model/anthropic.claude-sonnet-4-5-20250929-v1:0",
      "status": "ACTIVE"
    }
  ]
}
```

**Copy this ARN** - you'll need it later. The format is:
```
arn:aws:bedrock:REGION:ACCOUNT_ID:inference-profile/PROFILE_ID
```

### Step 4: Configure AWS Credentials

**What we're doing:** Setting up AWS CLI credentials so Claude Code can authenticate with your AWS account.

**Why this is separate from the console:** The AWS Console uses browser-based authentication, but CLI tools (like Claude Code) need programmatic access through access keys.

If you haven't configured AWS CLI yet:

```bash
aws configure --profile YOUR_AWS_PROFILE_NAME
```

You'll be prompted for:
- **AWS Access Key ID:** Found in AWS Console → IAM → Users → Security Credentials
- **AWS Secret Access Key:** Generated when you create the access key (save it securely!)
- **Default region:** e.g., `eu-west-2` (should match where you enabled model access)
- **Output format:** `json` (recommended)

**Verify your configuration works:**

```bash
aws sts get-caller-identity --profile YOUR_AWS_PROFILE_NAME
```

**Expected output:**
```json
{
    "UserId": "AIDACKCEVSQ6C2EXAMPLE",
    "Account": "ACCOUNT_ID",
    "Arn": "arn:aws:iam::ACCOUNT_ID:user/YourUserName"
}
```

**If this fails:** Check your credentials file:
- **Windows:** `C:\Users\USERNAME\.aws\credentials`
- **Mac/Linux:** `~/.aws/credentials`

**Reference:** [AWS CLI Configuration Guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

## Part 2: Testing AWS Bedrock Connection (Before Touching VS Code)

**Why this step saved me hours:** I initially jumped straight to configuring VS Code, which meant I was troubleshooting two things at once (AWS connection AND VS Code configuration). By testing the AWS connection separately first, I could confirm that part worked before moving on.

### Create a Test Request File

The AWS Bedrock API expects requests in a specific JSON format that follows the Anthropic Messages API structure.

**For PowerShell (Windows):**
```powershell
'{"anthropic_version":"bedrock-2023-05-31","max_tokens":1000,"messages":[{"role":"user","content":"Say hello"}]}' | Set-Content -Path request.json -NoNewline
```

**For Bash/Terminal (Mac/Linux):**
```bash
echo '{"anthropic_version":"bedrock-2023-05-31","max_tokens":1000,"messages":[{"role":"user","content":"Say hello"}]}' > request.json
```

**Breaking down this JSON:**
- `anthropic_version`: Required by Bedrock to specify the API version
- `max_tokens`: Maximum response length (1000 is reasonable for testing)
- `messages`: Array of conversation messages (Claude's standard format)
- `role`: Either "user" (your input) or "assistant" (Claude's responses)
- `content`: The actual message text

### Test the Bedrock API

**My PowerShell experience:** I initially tried inline JSON in PowerShell and kept getting "Invalid base64" errors. Turns out PowerShell handles quote escaping differently. Using a file (as shown above) solved this completely.

**For PowerShell (Windows):**
```powershell
aws bedrock-runtime invoke-model `
  --model-id arn:aws:bedrock:eu-west-2:ACCOUNT_ID:inference-profile/eu.anthropic.claude-sonnet-4-5-20250929-v1:0 `
  --body file://request.json `
  --region eu-west-2 `
  --profile YOUR_AWS_PROFILE_NAME `
  --cli-binary-format raw-in-base64-out `
  output.txt

Get-Content output.txt
```

**For Bash (Mac/Linux):**
```bash
aws bedrock-runtime invoke-model \
  --model-id arn:aws:bedrock:eu-west-2:ACCOUNT_ID:inference-profile/eu.anthropic.claude-sonnet-4-5-20250929-v1:0 \
  --body file://request.json \
  --region eu-west-2 \
  --profile YOUR_AWS_PROFILE_NAME \
  --cli-binary-format raw-in-base64-out \
  output.txt

cat output.txt
```

**Breaking down the command:**
- `bedrock-runtime invoke-model`: The Bedrock API endpoint for running inference
- `--model-id`: Your inference profile ARN (use the full ARN you found earlier)
- `--body file://request.json`: Path to your request JSON file
- `--region`: Must match your inference profile region
- `--profile`: Your AWS CLI profile name
- `--cli-binary-format raw-in-base64-out`: Tells AWS CLI to accept raw JSON input (not base64)
- `output.txt`: Where to save the response

### Expected Successful Response

If everything is configured correctly, `output.txt` should contain:

```json
{
  "model": "claude-sonnet-4-5-20250929",
  "id": "msg_bdrk_01VkZqvjyFoEp74SF26xipxW",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Hello! How can I help you today?"
    }
  ],
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 9,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0,
    "output_tokens": 12
  }
}
```

**What this tells us:**
- The model is responding (you got actual text back!)
- Your AWS credentials work
- Your inference profile is active
- You're being charged tokens (check the `usage` section)

**If you see an error here, stop!** Don't proceed to VS Code configuration until this works. See the Troubleshooting section below.

**Reference:** [AWS Bedrock Runtime API Documentation](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html)

## Part 3: Configuring VS Code Claude Code Extension (Where I Spent Most of My Time)

### Step 1: Open VS Code Settings

1. Open Visual Studio Code
2. Press `Ctrl+Shift+P` (Windows/Linux) or `Cmd+Shift+P` (Mac) to open the command palette
3. Type: `Preferences: Open User Settings (JSON)`
4. Press Enter

**Why JSON settings:** The Claude Code extension uses environment variables in a specific format that's easier to configure in JSON than in the GUI settings editor.

This opens your `settings.json` file, which might already have other VS Code configurations.

### Step 2: Add Claude Code Configuration (THE CRITICAL PART)

**My biggest mistake:** I initially configured `environmentVariables` as a simple object like this:

```json
// ❌ WRONG - This caused "e is not iterable" error
"claude-code.environmentVariables": {
    "AWS_PROFILE": "AWS_PROFILE_NAME",
    "AWS_REGION": "eu-west-2"
}
```

This seems logical - most configuration uses key-value objects. But Claude Code expects an **array of objects** with specific `name` and `value` properties.

**The correct configuration:**

Add this to your `settings.json` (or modify if these settings already exist):

```json
{
    "claude-code.selectedModel": "claude-sonnet-4-5-20250929",
    "claude-code.environmentVariables": [
        {
            "name": "AWS_PROFILE",
            "value": "AWS_PROFILE_NAME"
        },
        {
            "name": "AWS_REGION",
            "value": "eu-west-2"
        },
        {
            "name": "BEDROCK_MODEL_ID",
            "value": "arn:aws:bedrock:eu-west-2:ACCOUNT_ID:inference-profile/eu.anthropic.claude-sonnet-4-5-20250929-v1:0"
        },
        {
            "name": "CLAUDE_CODE_USE_BEDROCK",
            "value": "1"
        }
    ]
}
```

**Breaking down each setting:**

1. **`claude-code.selectedModel`**: 
   - The model identifier (without the full ARN)
   - This tells Claude Code which model to display in the UI
   - Use format: `claude-sonnet-4-5-20250929`

2. **`AWS_PROFILE`**:
   - Your AWS CLI profile name (from Step 4 in Part 1)
   - Must match exactly what's in your `~/.aws/credentials` file
   - Example: `AWS_PROFILE_NAME`

3. **`AWS_REGION`**:
   - The AWS region where you enabled model access
   - Must match your inference profile ARN region
   - Example: `eu-west-2`, `us-east-1`, `us-west-2`

4. **`BEDROCK_MODEL_ID`**:
   - The full inference profile ARN (from Step 3 in Part 1)
   - This is what Claude Code actually uses to call Bedrock
   - Format: `arn:aws:bedrock:REGION:ACCOUNT_ID:inference-profile/PROFILE_ID`

5. **`CLAUDE_CODE_USE_BEDROCK`**:
   - Tells Claude Code to use Bedrock instead of Anthropic's direct API
   - Must be set to `"1"` (as a string, not a number)

**Critical formatting rules:**
- ✅ `environmentVariables` MUST be an array: `[ ... ]`
- ✅ Each variable MUST be an object with `name` and `value` keys
- ✅ Both `name` and `value` must be strings (in quotes)
- ✅ All values should be exact matches (no extra spaces)
- ❌ Don't use a simple key-value object
- ❌ Don't forget the commas between array elements

### Step 3: Save and Reload VS Code

After saving your `settings.json`:

1. Press `Ctrl+Shift+P` (Windows/Linux) or `Cmd+Shift+P` (Mac)
2. Type: `Developer: Reload Window`
3. Press Enter

**Why reload:** VS Code needs to restart the Claude Code extension to pick up the new environment variables. Simply saving the file isn't enough.

**What happens during reload:**
- Claude Code extension reinitializes
- Environment variables are loaded
- The MCP (Model Context Protocol) server restarts
- New connections to Bedrock are established

### Step 4: Test Claude Code

**My experience:** After fixing the environment variables format, I opened Claude Code and held my breath. When I typed "say hello" and actually got a response back (instead of endless "thinking..." messages), I knew I'd finally figured it out!

1. Click the **Claude icon** in the VS Code sidebar (left side, looks like the Claude logo)
2. A panel opens with a text input at the bottom
3. Type a simple prompt: `say hello`
4. Press Enter

**What you should see:**

1. **Status messages** (these are normal!):
   - "thinking..."
   - "deliberating..."
   - "coalescing..."
   
2. **Actual response from Claude:**
   - A proper text response answering your question
   - The response appears in the chat window
   - You can continue the conversation

**Expected outcome:** Claude responds naturally to your prompts, just like in the web interface.

**If you only see status messages without responses:** See the Troubleshooting section below - this was my exact problem!

**Reference:** [Claude Code Documentation](https://docs.claude.com/en/docs/claude-code)

## Troubleshooting: Real Problems I Encountered (And How I Solved Them)

### Issue 1: "Invocation of model ID with on-demand throughput isn't supported"

**When this happened to me:** Right at the beginning, when I tried using the direct model ID instead of an inference profile.

**The error message:**
```
An error occurred (ValidationException) when calling the InvokeModel operation: 
Invocation of model ID anthropic.claude-sonnet-4-5-20250929-v1:0 with on-demand 
throughput isn't supported. Retry your request with the ID or ARN of an inference 
profile that contains this model.
```

**Root cause:** AWS Bedrock requires inference profiles for on-demand usage. You cannot use the model ID directly.

**Solution:** 
1. List available inference profiles:
   ```bash
   aws bedrock list-inference-profiles --region eu-west-2 --profile YOUR_AWS_PROFILE_NAME
   ```
2. Use the full inference profile ARN:
   ```
   arn:aws:bedrock:eu-west-2:ACCOUNT_ID:inference-profile/eu.anthropic.claude-sonnet-4-5-20250929-v1:0
   ```

**Why this matters:** Inference profiles provide better reliability and cross-region failover. AWS requires them for on-demand usage.

### Issue 2: "Malformed input request, please reformat your input and try again"

**When this happened to me:** Testing the AWS CLI with inline JSON on PowerShell.

**The error message:**
```
An error occurred (ValidationException) when calling the InvokeModel operation: 
Malformed input request, please reformat your input and try again.
```

**Root cause:** PowerShell handles JSON string escaping differently than bash, causing the JSON to be malformed by the time it reaches AWS.

**Solution:** Always use a file for the request body:

```powershell
# Create the file properly
'{"anthropic_version":"bedrock-2023-05-31","max_tokens":100,"messages":[{"role":"user","content":"Hello"}]}' | Set-Content -Path request.json -NoNewline

# Then reference it
aws bedrock-runtime invoke-model --body file://request.json ...
```

**Pro tip:** The `-NoNewline` parameter is important - it prevents PowerShell from adding an extra newline character that could cause parsing issues.

### Issue 3: "Invalid base64" Error

**When this happened to me:** Using inline JSON with PowerShell without the proper AWS CLI flag.

**The error message:**
```
Invalid base64: "{"anthropic_version":"bedrock-2023-05-31",...}"
```

**Root cause:** By default, AWS CLI expects the `--body` parameter to be base64-encoded. When you provide raw JSON, it tries to decode it as base64 and fails.

**Solution:** Add the `--cli-binary-format raw-in-base64-out` flag:

```powershell
aws bedrock-runtime invoke-model `
  --model-id YOUR_MODEL_ARN `
  --body file://request.json `
  --cli-binary-format raw-in-base64-out `
  ...
```

**What this flag does:**
- `raw-in`: Accept raw (non-base64) input
- `base64-out`: Return base64-encoded output (standard for binary data)

### Issue 4: Status Messages Only - No Responses (MY BIGGEST HEADACHE)

**When this happened to me:** After "successfully" configuring VS Code. Claude Code would open, I'd type a prompt, see "thinking...", "deliberating...", "coalescing...", and then... nothing. Just those status messages forever.

**What I saw in Developer Console:**
```
ERR e is not iterable: TypeError: e is not iterable
    at Wl (c:\Users\Vasko\.vscode\extensions\anthropic.claude-code-2.0.14\extension.js:69:3022)
```

**Root cause:** I used the wrong format for `environmentVariables` in `settings.json`. I used a simple object instead of an array of objects.

**My wrong configuration:**
```json
// ❌ THIS DOESN'T WORK
"claude-code.environmentVariables": {
    "AWS_PROFILE": "YOUR_AWS_PROFILE_NAME",
    "AWS_REGION": "eu-west-2",
    "BEDROCK_MODEL_ID": "arn:aws:bedrock:...",
    "CLAUDE_CODE_USE_BEDROCK": "1"
}
// ❌ THIS DOESN'T WORK AS WELL
"claude-code.environmentVariables": [
    "AWS_PROFILE=YOUR_AWS_PROFILE_NAME",
    "AWS_REGION=eu-west-2",
    "BEDROCK_MODEL_ID=arn:aws:bedrock:...",
    "CLAUDE_CODE_USE_BEDROCK=1"
]
```

**The correct configuration:**
```json
// ✅ THIS WORKS
"claude-code.environmentVariables": [
    {
        "name": "AWS_PROFILE",
        "value": "YOUR_AWS_PROFILE_NAME"
    },
    {
        "name": "AWS_REGION",
        "value": "eu-west-2"
    },
    {
        "name": "BEDROCK_MODEL_ID",
        "value": "arn:aws:bedrock:eu-west-2:ACCOUNT_ID:inference-profile/eu.anthropic.claude-sonnet-4-5-20250929-v1:0"
    },
    {
        "name": "CLAUDE_CODE_USE_BEDROCK",
        "value": "1"
    }
]
```

**Why this format:** The Claude Code extension internally iterates over the environment variables array. When it receives an object instead, JavaScript throws "e is not iterable" because you can't iterate over object keys the same way you iterate over arrays.

**Debugging steps that helped me:**

1. **Check Output Logs:**
   - View → Output
   - Select "Claude Code" from dropdown
   - Look for error messages when sending prompts

2. **Check Developer Console:**
   - Help → Toggle Developer Tools
   - Console tab shows JavaScript errors
   - "e is not iterable" was the smoking gun

3. **Verify the format:**
   - Check that `environmentVariables` starts with `[` not `{`
   - Each variable should have both `name` and `value` keys
   - All strings should be in quotes

4. **Reload after changes:**
   - Always reload VS Code window after changing settings
   - Cmd/Ctrl + Shift + P → "Developer: Reload Window"

### Issue 5: Authentication Errors / Access Denied

**Symptoms:** Errors like "Access Denied", "UnauthorizedException", or "Could not verify AWS credentials"

**Possible causes:**

1. **AWS profile doesn't exist:**
   ```bash
   # Check if your profile is configured
   aws configure list --profile YOUR_AWS_PROFILE_NAME
   ```

2. **Credentials file is missing or corrupted:**
   - **Windows:** Check `C:\Users\USERNAME\.aws\credentials`
   - **Mac/Linux:** Check `~/.aws/credentials`
   
   Should look like:
   ```
   [YOUR_AWS_PROFILE_NAME]
   aws_access_key_id = AKIAIOSFODNN7EXAMPLE
   aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   ```

3. **IAM permissions are insufficient:**
   Your AWS user needs these permissions:
   - `bedrock:InvokeModel`
   - `bedrock:ListFoundationModels`
   - `bedrock:ListInferenceProfiles`
   
   Check in AWS Console → IAM → Users → Your User → Permissions

4. **Profile name mismatch:**
   The profile name in VS Code settings must EXACTLY match the name in `~/.aws/credentials`
   - Check for typos
   - Check for extra spaces
   - Names are case-sensitive

**Verification command:**
```bash
# This should return your user info
aws sts get-caller-identity --profile YOUR_AWS_PROFILE_NAME
```

### Issue 6: Region Mismatch Errors

**Symptoms:** Model not found, or "requested resource is not available in this region"

**Root cause:** Inconsistency between where you enabled model access and where you're trying to use it.

**Three places that must match:**

1. **AWS Bedrock Console:** Where you enabled model access
   - Check: AWS Console → Bedrock → Model access
   - Note the region in the top-right corner

2. **Inference Profile ARN:** The region in your ARN
   ```
   arn:aws:bedrock:eu-west-2:...
                    ^^^^^^^^^^
                    This region
   ```

3. **VS Code Settings:** The `AWS_REGION` environment variable
   ```json
   {
       "name": "AWS_REGION",
       "value": "eu-west-2"  // Must match the above
   }
   ```

**Solution:** Pick one region and use it consistently everywhere. Popular choices:
- `us-east-1` (North Virginia) - Usually cheapest, most features
- `us-west-2` (Oregon) - West coast US
- `eu-west-2` (London) - European data residency
- `ap-southeast-1` (Singapore) - Asia Pacific

**Pro tip:** Different regions have different model availability and pricing. Check the [AWS Bedrock pricing page](https://aws.amazon.com/bedrock/pricing/) for your region.

## Verification Checklist

Before reaching out for support, verify each of these items:

### AWS Bedrock Setup
- [ ] Model access is enabled in AWS Bedrock console for Claude Sonnet 4.5
- [ ] Model status shows "Access granted" (not "Pending" or "Not enabled")
- [ ] You're using an inference profile ARN, not a direct model ID
- [ ] The inference profile is in "ACTIVE" status
- [ ] Your AWS region is consistent across all configurations

### AWS CLI Testing
- [ ] AWS CLI is installed and available in your terminal
- [ ] `aws sts get-caller-identity` works with your profile
- [ ] You can successfully invoke the model using the CLI test (Part 2)
- [ ] The CLI test returns actual JSON response with Claude's message

### VS Code Configuration
- [ ] Claude Code extension is installed and up to date
- [ ] `settings.json` uses the correct environment variables format (array of objects)
- [ ] Each environment variable has both `name` and `value` keys
- [ ] AWS profile name exactly matches your `~/.aws/credentials` file
- [ ] The `BEDROCK_MODEL_ID` is the full inference profile ARN
- [ ] `CLAUDE_CODE_USE_BEDROCK` is set to `"1"` (string, not number)
- [ ] VS Code has been reloaded after making configuration changes

### Debugging
- [ ] Check Claude Code output logs (View → Output → "Claude Code")
- [ ] Check Developer Console for errors (Help → Toggle Developer Tools)
- [ ] No "e is not iterable" errors in console
- [ ] No AWS authentication errors in logs

## Advanced Configuration Options

### Using Different Models

To use different Claude models, update these two settings:

1. **In VS Code settings.json:**
   ```json
   "claude-code.selectedModel": "claude-opus-4-20250514"
   ```

2. **In BEDROCK_MODEL_ID environment variable:**
   ```json
   {
       "name": "BEDROCK_MODEL_ID",
       "value": "arn:aws:bedrock:eu-west-2:ACCOUNT_ID:inference-profile/eu.anthropic.claude-opus-4-20250514-v1:0"
   }
   ```

Available models:
- `claude-sonnet-4-5-20250929` - Best balance of intelligence and speed
- `claude-opus-4-20250514` - Most capable, slower and more expensive
- `claude-haiku-4-20250107` - Fastest and cheapest, good for simple tasks

### Multiple AWS Profiles

If you work with multiple AWS accounts, you can create separate VS Code workspace settings:

1. Open your workspace folder in VS Code
2. Create `.vscode/settings.json` in your project root
3. Add Claude Code configuration specific to that project
4. Workspace settings override user settings

This allows different projects to use different AWS accounts or regions.

### Environment Variables Priority

Claude Code checks for environment variables in this order:
1. VS Code `settings.json` (what we configured)
2. System environment variables
3. `.env` file in workspace root

If you prefer system-level configuration, you can set these environment variables in your OS instead of VS Code settings.

## Additional Resources

### Official Documentation
- **Claude Code Extension:** https://docs.claude.com/en/docs/claude-code
- **AWS Bedrock User Guide:** https://docs.aws.amazon.com/bedrock/latest/userguide/
- **AWS Bedrock API Reference:** https://docs.aws.amazon.com/bedrock/latest/APIReference/
- **Anthropic Messages API:** https://docs.anthropic.com/en/api/messages

### AWS Bedrock Specific
- **Model Access:** https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html
- **Inference Profiles:** https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html
- **IAM Permissions:** https://docs.aws.amazon.com/bedrock/latest/userguide/security_iam_id-based-policy-examples.html
- **Bedrock Pricing:** https://aws.amazon.com/bedrock/pricing/

### Getting Help
- **Claude Support:** https://support.claude.com
- **AWS Support:** https://console.aws.amazon.com/support/
- **VS Code Issues:** https://github.com/microsoft/vscode/issues

## Conclusion: What I Learned

After several hours of troubleshooting, here are my key takeaways:

1. **Test AWS connectivity first:** Don't try to debug VS Code and AWS at the same time. Get the AWS CLI working first, then move to VS Code.

2. **The environment variables format is critical:** This was my biggest stumbling block. It must be an array of objects with `name` and `value` properties, not a simple key-value object.

3. **Inference profiles are mandatory:** You can't use direct model IDs with on-demand throughput. Always use the full inference profile ARN.

4. **Developer Console is your friend:** When Claude Code isn't working, the first place to look is Help → Toggle Developer Tools. Error messages there saved me hours.

5. **Region consistency matters:** Make sure your model access, inference profile, and environment variables all use the same AWS region.

The initial setup has a learning curve, but once configured correctly, Claude Code with AWS Bedrock provides a powerful, secure way to integrate AI-assisted coding into your development workflow.

## Real-World Usage Tips (What I Wish I Knew Earlier)

### Understanding Token Usage and Costs

**My surprise:** After getting everything working, I was excited to use Claude Code extensively. Then I checked my AWS bill and realized I needed to understand token consumption better.

**How Bedrock charges:**
- **Input tokens:** Everything you send to Claude (your code, prompts, context)
- **Output tokens:** Everything Claude generates back
- **Pricing varies by model and region**

For Claude Sonnet 4.5 (as of September 2025):
- Input: $3 per million tokens
- Output: $15 per million tokens

**Rough token estimates:**
- 1 token ≈ 4 characters of English text
- 1 token ≈ ¾ of a word
- 100 tokens ≈ 75 words
- 1,000 tokens ≈ 750 words (approximately 1.5-3 standard pages)

*Note: AWS Bedrock pricing may include additional AWS service charges. Check the AWS Bedrock pricing page for your specific region.*

**Cost management tips:**

1. **Be specific with prompts:** Don't send entire file contents if you only need Claude to look at specific functions
2. **Use workspace context wisely:** Claude Code can read your entire workspace, but that uses tokens
3. **Monitor usage:** Check CloudWatch or AWS Cost Explorer regularly
4. **Set budget alerts:** Configure AWS Budget alerts to notify you at spending thresholds

**Reference:** [AWS Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)

### Using Claude Code Effectively

**Best practices I discovered:**

1. **Start with specific files:** Use "Add files to context" to limit what Claude sees
2. **Clear context regularly:** Start new conversations for unrelated tasks
3. **Use the terminal integration:** Claude Code can suggest terminal commands - very powerful!
4. **Review suggestions carefully:** Always review code changes before accepting them
5. **Iterative refinement:** Ask Claude to refine its suggestions rather than starting over

**Example workflow that works well:**
```
You: "Look at src/utils/parser.js and explain what it does"
[Claude analyzes just that file]

You: "Now add error handling for null inputs"
[Claude provides specific changes]

You: "Add unit tests for those error cases"
[Claude generates tests]
```

### Workspace Setup for Better Results

**What helped me get better responses:**

1. **Good project structure:** Claude works better with organized codebases
2. **Clear file naming:** Descriptive filenames help Claude understand context
3. **README files:** A good README helps Claude understand your project faster
4. **Type definitions:** If using TypeScript, Claude leverages types effectively

### Security Considerations

**Important things I learned about security:**

1. **Credentials in code:** Never commit AWS credentials. Claude Code reads from `~/.aws/credentials` - keep that secure
2. **Sensitive data:** Be careful what files you let Claude Code access. If your workspace contains secrets, API keys, or sensitive data, Claude will see them
3. **IAM least privilege:** Give your AWS user only the Bedrock permissions it needs:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "bedrock:InvokeModel",
                   "bedrock:ListFoundationModels"
               ],
               "Resource": "*"
           }
       ]
   }
   ```
4. **VPC endpoints:** For extra security, you can configure Bedrock to use VPC endpoints (keeping traffic within AWS)

**Reference:** [AWS Bedrock Security Best Practices](https://docs.aws.amazon.com/bedrock/latest/userguide/security-best-practices.html)

### Performance Optimization

**Tips for faster responses:**

1. **Use Haiku for simple tasks:** Claude Haiku is much faster and cheaper for straightforward questions
2. **Limit context size:** The more context Claude processes, the longer it takes
3. **Be specific:** "Fix the bug in line 45" is faster than "review this entire file"
4. **Cross-region profiles help:** They automatically route to less-busy regions

### Handling Common Scenarios

**Scenario 1: Working with Large Codebases**

Problem: Claude Code tries to read too many files, causing slow responses or token limit errors.

Solution:
- Use `.claudeignore` file (similar to `.gitignore`)
- Explicitly add only relevant files to context
- Break large refactoring tasks into smaller chunks

**Scenario 2: Inconsistent Responses**

Problem: Sometimes Claude gives great suggestions, sometimes they're off-base.

Solution:
- Provide more context in your prompt
- Show Claude examples of your coding style
- Reference specific functions or patterns you want to follow
- Use "Please follow the pattern in [filename]" type prompts

**Scenario 3: Claude Suggests Changes to Wrong Files**

Problem: Claude modifies files you didn't intend.

Solution:
- Clear context before starting new tasks
- Be explicit: "Only modify src/utils/helper.js"
- Review the file list Claude is considering before approving changes

## Troubleshooting After Setup

### Issue 7: Slow Response Times

**When I noticed this:** After a few days of usage, responses started taking 30+ seconds.

**Possible causes:**
1. **Large workspace context:** Claude is reading too many files
2. **Complex prompts:** Very detailed or multi-part questions take longer
3. **Region congestion:** Your chosen region might be under heavy load
4. **Token limits:** Approaching context window limits causes slowdowns

**Solutions:**
- Clear Claude's context (start a new conversation)
- Use more specific file selections
- Try a different time of day
- Switch to cross-region inference profile for automatic routing
- Use Claude Haiku for simpler tasks

### Issue 8: "Context Window Exceeded" Errors

**The error:**
```
Error: This request would exceed the model's context window
```

**What this means:** Claude has a maximum context window (amount of text it can process at once). For Claude Sonnet 4.5, that's 200,000 tokens (~150,000 words).

**Solutions:**
1. **Reduce context:** Remove files from Claude's context
2. **Summarize previous conversation:** Start a new conversation with a summary
3. **Break up the task:** Split large refactoring into smaller pieces
4. **Use selective file inclusion:** Don't let Claude read your entire codebase

### Issue 9: Unexpected AWS Charges

**When this happened:** My first month's AWS bill was higher than expected.

**Common causes:**
1. **Token-heavy tasks:** Repeatedly asking Claude to analyze large files
2. **Leaving Claude Code open:** Background processes may consume tokens
3. **Not clearing context:** Long conversation threads accumulate token usage
4. **Using Opus unnecessarily:** Opus is 3-5x more expensive than Sonnet

**Prevention:**
1. **Set up AWS Budget alerts:**
   - AWS Console → Billing → Budgets
   - Create a budget with email notifications
   - Example: Alert at $50, $100, $150

2. **Monitor in CloudWatch:**
   - Track `InvocationCount` and `TokenUsage` metrics
   - Set up custom dashboards

3. **Use Cost Explorer:**
   - AWS Console → Cost Explorer
   - Filter by "Bedrock" service
   - View daily/monthly trends

4. **Optimize usage:**
   - Use Haiku for simple tasks
   - Use Sonnet for most development work
   - Reserve Opus for complex reasoning tasks only

**Reference:** [AWS Cost Management](https://docs.aws.amazon.com/cost-management/)

### Issue 10: Extension Updates Break Configuration

**What happened:** After updating Claude Code extension, my configuration stopped working.

**Why:** Extension updates sometimes change configuration formats or requirements.

**Solution:**
1. **Check release notes:** Read what changed in the update
2. **Verify settings format:** Ensure your `settings.json` still uses the correct format
3. **Check extension logs:** View → Output → "Claude Code" for migration messages
4. **Downgrade if necessary:** Extensions → Claude Code → gear icon → Install Another Version

**Pro tip:** Before updating, copy your working `settings.json` configuration somewhere safe.

## Going Further: Advanced Integration

### Integrating with CI/CD

You can use AWS Bedrock programmatically in your CI/CD pipelines:

```yaml
# Example GitHub Actions workflow
name: AI Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-2
      - name: AI Review
        run: |
          # Your script that calls Bedrock API
          python ai_review.py
```

### Using with Other Tools

Claude Code works alongside other VS Code extensions:

- **GitHub Copilot:** Use both! They complement each other
- **ESLint/Prettier:** Claude respects your linting rules
- **Debugger:** Ask Claude to help interpret error messages
- **Git:** Claude can help write commit messages and review diffs

### Custom Prompts and Templates

Create reusable prompt templates for common tasks:

**Example: Code Review Template**
```
Please review this code for:
1. Security vulnerabilities
2. Performance issues
3. Best practices violations
4. Potential bugs

File: [filename]
Focus areas: [specific concerns]
```

Save these in your workspace for consistent reviews.

### Team Adoption

**Tips for introducing Claude Code to your team:**

1. **Start with documentation:** This guide is a starting point
2. **Shared AWS account:** Set up a team AWS account with Bedrock access
3. **Cost allocation tags:** Tag resources by team/project for better cost tracking
4. **Internal best practices:** Document what works for your codebase
5. **Regular reviews:** Check how the team is using it, optimize together

## My Final Thoughts

Getting Claude Code working with AWS Bedrock took me several hours of troubleshooting, but it was absolutely worth it. The combination of:
- Claude's powerful AI capabilities
- AWS Bedrock's enterprise security
- VS Code's development environment
- Existing AWS infrastructure

...creates an incredibly productive development experience.

**The main lessons from my journey:**

1. **Don't skip the testing phase:** Verify AWS connectivity before touching VS Code
2. **Pay attention to data structures:** That array-vs-object difference cost me hours
3. **Read error messages carefully:** "e is not iterable" was telling me exactly what was wrong
4. **Use the developer tools:** Console logs revealed issues that weren't obvious otherwise
5. **Document as you go:** I wish I'd taken notes during troubleshooting

**What I use Claude Code for now:**
- Explaining unfamiliar codebases
- Writing unit tests
- Refactoring complex functions
- Debugging obscure errors
- Generating boilerplate code
- Code reviews before committing

**What I still do myself:**
- Architecture decisions
- Security-critical code
- Final code review and testing
- Anything involving business logic

The key is finding the right balance - Claude Code is a powerful assistant, but you're still the developer making the final decisions.

## Keeping This Configuration Working

**Monthly maintenance:**
- Check AWS bills for unexpected usage
- Update Claude Code extension when new versions release
- Review AWS IAM permissions (remove unnecessary access)
- Clear old conversations in Claude Code to free up local storage

**When things break:**
1. Check if AWS credentials are still valid
2. Verify model access is still enabled in Bedrock
3. Check for Claude Code extension updates
4. Review VS Code settings.json for changes
5. Test AWS CLI connection first

**Staying informed:**
- Subscribe to AWS Bedrock updates
- Follow Claude Code release notes
- Join relevant developer communities

## Acknowledgments

This guide was born out of frustration, trial and error, and eventual success. I hope it saves you the hours of troubleshooting I went through.

Special thanks to:
- The Claude Code team for building this powerful extension
- AWS Bedrock documentation (even though I didn't read it carefully enough at first!)
- The developer community for shared troubleshooting experiences
- My patience, which was tested but ultimately rewarded

If this guide helped you, pay it forward - help the next developer struggling with their configuration!

---

**Document Version:** 1.0  
**Last Updated:** October 2025  
**Author's Setup:** VS Code on Windows, AWS Bedrock in eu-west-2, Claude Code Extension 2.0.14  

**Feedback?** If you find errors or have suggestions to improve this guide, I'd love to hear them. The best documentation is the documentation that helps real developers solve real problems.