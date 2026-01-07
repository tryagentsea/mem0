# Google Cloud Build Deployment Guide

This guide explains how to deploy the mem0 server to Google Cloud Run using Cloud Build.

## Prerequisites

1. **Google Cloud Project** with billing enabled
2. **gcloud CLI** installed and authenticated
3. **Required APIs enabled**:
   ```bash
   gcloud services enable cloudbuild.googleapis.com
   gcloud services enable run.googleapis.com
   gcloud services enable containerregistry.googleapis.com
   gcloud services enable secretmanager.googleapis.com
   ```

## Setup

### 1. Create Environment File

Create a `.env` file with all your environment variables:

```bash
UPSTASH_VECTOR_REST_URL=https://your-upstash-url.upstash.io
UPSTASH_VECTOR_REST_TOKEN=your-upstash-token
GROQ_API_KEY=your-groq-api-key
API_AUTH_TOKEN=your-api-auth-token
```

### 2. Store in Secret Manager

Upload the entire `.env` file as a single secret:

```bash
gcloud secrets create mem0-env \
  --data-file=.env \
  --replication-policy="automatic"
```

### 3. Grant Permissions

Grant Cloud Build and Cloud Run access to the secret:

```bash
# Get your project number
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")

# Grant Cloud Build service account access
gcloud secrets add-iam-policy-binding mem0-env \
  --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Grant Cloud Run service account access
gcloud secrets add-iam-policy-binding mem0-env \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

## Deployment

### Manual Deployment

From the **repository root** directory (parent of `server/`):

```bash
export PROJECT_ID="your-gcp-project-id"

gcloud builds submit \
  --config=server/cloudbuild.yaml \
  --project=$PROJECT_ID
```

That's it! All secrets are automatically pulled from Secret Manager.

### Automated Deployment with Triggers

Create a Cloud Build trigger for automatic deployments:

```bash
gcloud builds triggers create github \
  --name="mem0-server-deploy" \
  --repo-name="YOUR_REPO_NAME" \
  --repo-owner="YOUR_GITHUB_USERNAME" \
  --branch-pattern="^main$" \
  --build-config="server/cloudbuild.yaml"
```

Or via the Cloud Console:
1. Go to **Cloud Build > Triggers**
2. Click **Create Trigger**
3. Connect your repository
4. Set **Branch**: `^main$`
5. Set **Build Configuration**: `server/cloudbuild.yaml`
6. Click **Create**

## Configuration

### Environment Variables (Stored in Secret Manager)

All configuration is stored in a single `.env` file in Secret Manager as `mem0-env`:

| Variable | Description | Required |
|----------|-------------|----------|
| `UPSTASH_VECTOR_REST_URL` | Upstash Vector database URL | Yes |
| `UPSTASH_VECTOR_REST_TOKEN` | Upstash authentication token | Yes |
| `GROQ_API_KEY` | Groq API key for LLM | Yes |
| `API_AUTH_TOKEN` | Bearer token for API authentication | Yes |
| `HISTORY_DB_PATH` | SQLite history path (defaults to `:memory:`) | No |
| `MEM0_DIR` | Mem0 config directory (defaults to `/tmp/.mem0`) | No |

### Cloud Run Settings

Default configuration:
- **Region**: `us-central1` (change in `cloudbuild.yaml`)
- **Memory**: 1 GiB
- **CPU**: 1
- **Min instances**: 0 (scales to zero)
- **Max instances**: 10
- **Timeout**: 300s

Modify these in the `cloudbuild.yaml` file as needed.

## Customization

### Change Region

Edit `cloudbuild.yaml` and update the `--region` flag:

```yaml
- '--region'
- 'us-east1'  # Change to your preferred region
```

### Update Environment Variables

To update your environment variables:

1. Edit your local `.env` file
2. Update the secret in Secret Manager:

```bash
gcloud secrets versions add mem0-env --data-file=.env
```

That's it! The next deployment will automatically use the new values.

### Adjust Resources

Modify memory and CPU allocations:

```yaml
- '--memory'
- '2Gi'  # Increase memory
- '--cpu'
- '2'    # Increase CPU
```

### Enable Authentication

To require Google Cloud authentication for API access, change:

```yaml
- '--allow-unauthenticated'
```

to:

```yaml
- '--no-allow-unauthenticated'
```

## Monitoring

### View Logs

```bash
gcloud run services logs read mem0-server \
  --region=us-central1 \
  --limit=50
```

### Get Service URL

```bash
gcloud run services describe mem0-server \
  --region=us-central1 \
  --format="value(status.url)"
```

### View Service Details

```bash
gcloud run services describe mem0-server \
  --region=us-central1
```

## Testing

Once deployed, test your API:

```bash
# Get the service URL
SERVICE_URL=$(gcloud run services describe mem0-server \
  --region=us-central1 \
  --format="value(status.url)")

# Test with authentication
curl -H "Authorization: Bearer YOUR_API_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$SERVICE_URL/memories" \
  -d '{
    "messages": [
      {"role": "user", "content": "My name is Alice"}
    ],
    "user_id": "alice123"
  }'
```

## Troubleshooting

### Build Fails

Check build logs:
```bash
gcloud builds list --limit=5
gcloud builds log BUILD_ID
```

### Service Won't Start

Check service logs:
```bash
gcloud run services logs read mem0-server --region=us-central1
```

### Secret Access Issues

Verify service account has access:
```bash
gcloud secrets get-iam-policy mem0-env
```

## Cost Optimization

- **Scale to zero**: Set `--min-instances=0` (already default)
- **Use smaller instances**: Reduce memory/CPU if sufficient
- **Set request timeout**: Prevent long-running requests
- **Enable concurrency**: Allow multiple requests per instance

```yaml
- '--concurrency'
- '80'  # Default is 80, adjust based on workload
```

## Additional Resources

- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Cloud Build Documentation](https://cloud.google.com/build/docs)
- [Secret Manager Documentation](https://cloud.google.com/secret-manager/docs)

