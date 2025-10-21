# n8n Docker Setup for BCS Debt Collection Bot

This guide explains how to set up a new n8n instance with VAPI environment variables using Docker Compose.

## Why a New Container?

This setup creates a **separate n8n container** that:
- Runs on port **5679** (your existing container is on 5678)
- Uses a separate data directory (`~/.n8n-vapi`)
- Has all VAPI environment variables pre-configured
- Won't interfere with your existing n8n instance

## Prerequisites

- Docker and Docker Compose installed
- The VAPI assistant IDs (from running the Postman collections)
- Your VAPI API key

## Setup Instructions

### Step 1: Create Environment File

Copy the example environment file:

```bash
cp .env.example .env
```

### Step 2: Configure Environment Variables

Edit the `.env` file and fill in your actual values:

```bash
# Required - Get from VAPI Dashboard
VAPI_API_KEY=your_actual_vapi_api_key

# Required - The 6 assistant IDs you created via Postman
VAPI_CONSUMER_1=paste_consumer_1_id_here
VAPI_CONSUMER_2=paste_consumer_2_id_here
VAPI_CONSUMER_3=paste_consumer_3_id_here
VAPI_COMMERCIAL_1=paste_commercial_1_id_here
VAPI_COMMERCIAL_2=paste_commercial_2_id_here
VAPI_COMMERCIAL_3=paste_commercial_3_id_here

# Optional but recommended
VAPI_PHONE_NUMBER_ID=ph_your_phone_number_id
VAPI_WEBHOOK_SECRET=your_webhook_secret

# Payment details (read verbally by AI)
STRIPE_PAYMENT_LINK=https://buy.stripe.com/xxxxx
BSB_NUMBER=123-456
ACCOUNT_NUMBER=123456789
BCS_PHONE_NUMBER=+61xxxxxxxxx
BCS_MAILING_ADDRESS=123 Main St, Sydney NSW 2000
```

### Step 3: Start the Container

```bash
docker-compose up -d
```

### Step 4: Access n8n

Open your browser and navigate to:

**http://localhost:5679**

Your existing n8n instance will still be running at http://localhost:5678

### Step 5: Import Workflows

1. Log into the new n8n instance at http://localhost:5679
2. Go to **Workflows**
3. Click **Import from File**
4. Import the workflows from `New Solution/N8N/`:
   - `Debt.json`

### Step 6: Test Environment Variables

Create a test workflow with a **Code** node:

```javascript
return [
  {
    json: {
      VAPI_CONSUMER_1: $env.VAPI_CONSUMER_1,
      VAPI_CONSUMER_2: $env.VAPI_CONSUMER_2,
      VAPI_CONSUMER_3: $env.VAPI_CONSUMER_3,
      VAPI_API_KEY: $env.VAPI_API_KEY ? 'Set' : 'Not Set'
    }
  }
];
```

Run it to verify all environment variables are loaded correctly.

## Container Management

### View Logs
```bash
docker-compose logs -f
```

### Stop Container
```bash
docker-compose down
```

### Restart Container
```bash
docker-compose restart
```

### Stop and Remove Container (keeps data)
```bash
docker-compose down
```

### Stop and Remove Everything (including data)
```bash
docker-compose down -v
rm -rf ~/.n8n-vapi
```

## Migrating from Your Old Container

Once you've verified the new container works:

### Option 1: Export/Import Workflows
1. Export workflows from old container (http://localhost:5678)
2. Import into new container (http://localhost:5679)
3. Stop old container: `docker stop <old-container-name>`

### Option 2: Copy Data Directory
```bash
# Stop new container
docker-compose down

# Copy data from old instance
cp -r ~/.n8n/* ~/.n8n-vapi/

# Start new container
docker-compose up -d
```

## Accessing Environment Variables in n8n

In any n8n node, you can access environment variables using:

```javascript
$env.VAPI_CONSUMER_1
$env.VAPI_API_KEY
$env.BSB_NUMBER
```

Example in HTTP Request node:
- **URL**: `https://api.vapi.ai/assistant/{{$env.VAPI_CONSUMER_1}}`
- **Authentication**: Bearer Token
- **Token**: `{{$env.VAPI_API_KEY}}`

## Troubleshooting

### Port Already in Use
If port 5679 is already taken, edit `docker-compose.yml` and change:
```yaml
ports:
  - "5680:5678"  # Use 5680 instead
```

### Environment Variables Not Loading
1. Check `.env` file exists in the same directory as `docker-compose.yml`
2. Restart container: `docker-compose restart`
3. Check logs: `docker-compose logs -f`

### Can't Access n8n UI
1. Check container is running: `docker ps`
2. Check logs: `docker-compose logs -f n8n`
3. Try http://127.0.0.1:5679 instead of localhost

### Need to Update Assistant IDs
1. Edit `.env` file with new IDs
2. Restart: `docker-compose restart`

## Production Deployment

For production deployment:

1. Change port to 80 or use reverse proxy (nginx)
2. Set up SSL/TLS certificates
3. Update `WEBHOOK_URL` in `.env` to your public domain
4. Configure VAPI webhook URL in dashboard to point to your server
5. Set strong passwords
6. Use Docker secrets for sensitive data

## Next Steps

After setup:
1. Import the Debt.json workflow
2. Configure webhook nodes with your VAPI webhook secret
3. Test with a sample debtor record
4. Monitor logs during test calls
5. Verify call outcomes are logged correctly
