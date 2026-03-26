# GitHub REST API — File Creation and Commits

This document covers the GitHub REST API endpoints needed to create a file in a repository and commit it. All endpoints use the base URL `https://api.github.com`.

## Authentication

All requests require authentication via a personal access token in the `Authorization` header:

```
Authorization: Bearer ghp_your_token_here
```

All requests should include:

```
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2022-11-28
```

## Option A: Contents API (Simple — Single File Create + Commit)

The Contents API creates a file and a commit in a single call. This is the simplest approach for creating or updating a single file.

### Create a File

```
PUT /repos/{owner}/{repo}/contents/{path}
```

**Path Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `owner` | string | Yes | Repository owner (username or org) |
| `repo` | string | Yes | Repository name |
| `path` | string | Yes | Path to the file in the repository (e.g., `test-experiment.md`) |

**Request Body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string | Yes | The commit message |
| `content` | string | Yes | The file content, encoded in **Base64** |
| `branch` | string | No | Branch name. Defaults to the repo's default branch. |

**Example Request**

```bash
curl -X PUT \
  -H "Authorization: Bearer ghp_your_token_here" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  -d '{
    "message": "Add experiment test file",
    "content": "TУНQ IHZzIENMSSB2cyBBUEkgZXhwZXJpbWVudA=="
  }' \
  "https://api.github.com/repos/{owner}/{repo}/contents/test-experiment.md"
```

> **Note:** The `content` field must be Base64-encoded. For example, "MCP vs CLI vs API experiment" encodes to `TUNQIHZZIENMSsb2cyBBUEkgZXhwZXJpbWVudA==`.

**Response (201 Created)**

```json
{
  "content": {
    "name": "test-experiment.md",
    "path": "test-experiment.md",
    "sha": "abc123...",
    "size": 31,
    "type": "file"
  },
  "commit": {
    "sha": "def456...",
    "message": "Add experiment test file",
    "author": {
      "name": "Your Name",
      "email": "you@example.com",
      "date": "2026-03-26T12:00:00Z"
    }
  }
}
```

**Error Responses**

| Status | Scenario | Resolution |
|--------|----------|------------|
| `404` | Repository not found or no access | Verify owner, repo name, and token permissions |
| `409` | File already exists at that path | Use the update flow (include the existing file's `sha` in the request body) |
| `422` | Invalid request (e.g., content not Base64) | Verify content is properly Base64-encoded |

## Option B: Git Database API (Low-Level — Full Control)

The Git Database API provides lower-level control over the commit process. This is useful when creating multiple files in a single commit or when you need precise control over the tree and commit objects.

### Step 1: Get the Current Reference

Get the SHA of the latest commit on the target branch.

```
GET /repos/{owner}/{repo}/git/ref/heads/{branch}
```

**Response (200 OK)**

```json
{
  "ref": "refs/heads/main",
  "object": {
    "sha": "abc123...",
    "type": "commit"
  }
}
```

### Step 2: Create a Blob

Create a blob object containing the file content.

```
POST /repos/{owner}/{repo}/git/blobs
```

**Request Body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `content` | string | Yes | The file content (raw text or Base64) |
| `encoding` | string | Yes | `"utf-8"` for raw text or `"base64"` for Base64-encoded content |

**Response (201 Created)**

```json
{
  "sha": "blob_sha_here",
  "url": "https://api.github.com/repos/{owner}/{repo}/git/blobs/blob_sha_here"
}
```

### Step 3: Create a Tree

Create a tree that includes the new file, using the current commit's tree as the base.

```
POST /repos/{owner}/{repo}/git/trees
```

**Request Body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `base_tree` | string | Yes | The SHA of the tree from the current commit |
| `tree` | array | Yes | Array of tree entry objects |

**Tree Entry Object**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | string | Yes | File path in the repository |
| `mode` | string | Yes | `"100644"` for a regular file |
| `type` | string | Yes | `"blob"` for a file |
| `sha` | string | Yes | The SHA of the blob created in Step 2 |

**Response (201 Created)**

```json
{
  "sha": "tree_sha_here"
}
```

### Step 4: Create a Commit

Create a new commit pointing to the tree created in Step 3.

```
POST /repos/{owner}/{repo}/git/commits
```

**Request Body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string | Yes | The commit message |
| `tree` | string | Yes | The SHA of the tree from Step 3 |
| `parents` | string[] | Yes | Array containing the SHA of the parent commit (from Step 1) |

**Response (201 Created)**

```json
{
  "sha": "commit_sha_here",
  "message": "Add experiment test file"
}
```

### Step 5: Update the Reference

Move the branch pointer to the new commit.

```
PATCH /repos/{owner}/{repo}/git/refs/heads/{branch}
```

**Request Body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sha` | string | Yes | The SHA of the new commit from Step 4 |

**Response (200 OK)**

```json
{
  "ref": "refs/heads/main",
  "object": {
    "sha": "commit_sha_here"
  }
}
```

## Token Permissions

The personal access token needs the following permissions:

- **Fine-grained tokens:** `Contents: Read and write` on the target repository
- **Classic tokens:** `repo` scope
