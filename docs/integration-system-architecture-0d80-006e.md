# Integration System

This document outlines the integration system architecture, how to use existing integrations, and how to add new integration providers.

## Overview

The integration system allows users to connect external services (like GitHub, GitLab, etc.) to the application. These integrations enable the application to:

- Fetch data from external services
- Sync content (issues, PRs, repositories, documentation)
- Keep information up-to-date
- Monitor changes in external systems

## Architecture

The integration system follows a service-based pattern:

### Server-side
- Base `IntegrationService` class that defines common operations
- Provider-specific implementations (e.g., `GitHubService`)
- Factory pattern via `createIntegrationService()` to instantiate the correct service
- Database storage of integration configurations and tokens
- Synchronization service for background processing

### Client-side
- TanStack Query + tRPC for API communication
- Popup-based OAuth flow
- Generic callback handling
- PostMessage communication between windows

## Database Schema

The integration system uses several related models in the database:

- `Integration`: Stores integration configurations and authentication data
- `Source`: Represents content imported from integrations
- `SourceChunk`: Contains processed chunks of source content
- `ChangedFile`: Tracks files modified in PRs or commits

The `Integration` model includes:
```prisma
model Integration {
  id          String          @id @default(cuid())
  name        String
  type        IntegrationType
  workspaceId String
  workspace   Workspace       @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  config      Json?           @db.JsonB // Configuration for the integration
  metadata    Json?           @db.JsonB // Additional metadata
  status      String          @default("active") // active, inactive, error
  sources     Source[] // All sources from this integration
  createdAt   DateTime        @default(now())
  updatedAt   DateTime        @updatedAt

  // Change Monitoring Configuration
  monitorChanges     Boolean  @default(false)
  webhookSecret      String?
  pollingEnabled     Boolean  @default(false)
  pollingInterval    Int?     // In minutes
  lastPolledAt       DateTime?
  nextPollAt         DateTime?
  
  // Monitoring stats
  changeCount        Int      @default(0)
  lastError          String?
  errorCount         Int      @default(0)
  
  // Relationship to feedback signals
  feedbackSignals    FeedbackSignal[]
  
  // every workspace can only have one integration of a given type
  @@unique([type, workspaceId])
  @@index([workspaceId])
}
```

## Available Integrations

Currently, the following integrations are available:

| Provider | Status | Features |
|----------|--------|----------|
| GitHub   | Available | Issues, PRs, Repositories, Code |
| GitLab   | Planned | Issues, MRs, Code |
| Jira     | Planned | Issues, Projects |
| Slack    | Available | Messages, Webhooks |
| Google Drive | Available | Files, Google Docs, Sheets, Slides |
| Docusaurus | Available | Documentation |
| VitePress | Available | Documentation |
| GitBook  | Available | Documentation |

## Environment Configuration

Each integration requires specific environment variables to be set:

### GitHub

```
# GitHub OAuth App credentials
GITHUB_CLIENT_ID=your_client_id
GITHUB_CLIENT_SECRET=your_client_secret
GITHUB_INTEGRATION_REDIRECT_URI=http://localhost:5173/auth-callback?provider=GitHub
```

### Slack

```
# Slack App credentials
SLACK_CLIENT_ID=your_slack_app_client_id
SLACK_CLIENT_SECRET=your_slack_app_client_secret
SLACK_SIGNING_SECRET=your_slack_signing_secret
```

### Google Drive

```
# Google Drive OAuth App credentials
GOOGLE_DRIVE_CLIENT_ID=your_google_oauth_app_client_id
GOOGLE_DRIVE_CLIENT_SECRET=your_google_oauth_app_client_secret
```

## OAuth Flow

The integration system uses a standardized OAuth flow:

1. User clicks "Connect" for an integration
2. A popup window opens with the OAuth provider's authorization page
3. User authorizes the application
4. Provider redirects to our callback URL with auth code
5. The callback exchanges the code for an access token
6. Token is stored in the database with the integration configuration
7. Popup window closes automatically, notifying the parent window
8. Parent window refreshes the integrations list

### Implementation Details

The system uses Redis to store temporary OAuth state to prevent CSRF attacks:

```typescript
// Generate state and store it in Redis
const state = crypto.randomBytes(20).toString('hex');
RedisService.set(`integrationState:${state}`, JSON.stringify({
  providerType: input.providerType,
  features: input.features || ['issues', 'prs', 'code'],
  redirectUrl: input.redirectUrl,
  workspaceId: input.workspaceId,
}));
```

On callback, the code is exchanged for a token and stored in the database:

```typescript
const tokenResponse = await fetch('https://github.com/login/oauth/access_token', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json'
  },
  body: JSON.stringify({
    client_id: githubEnv.github.clientId,
    client_secret: githubEnv.github.clientSecret,
    code: input.code,
    redirect_uri: redirectUrl
  })
});
```

## Adding a New Integration Provider

To add a new integration provider, follow these steps:

### 1. Create a Service Class

Create a new file in `server/src/services/integrations/` named `[provider].service.ts`:

```typescript
import { Integration, IntegrationConfig, IntegrationService } from './integration.service';

// Define provider-specific configuration
export interface ProviderConfig extends IntegrationConfig {
  token: string;
  // Add other provider-specific configuration
}

export class ProviderService extends IntegrationService {
  private token: string;
  
  constructor(integration: Integration) {
    super();
    if (!integration) {
      this.token = '';
      return;
    }
    const config = integration.config ? (integration.config as unknown as ProviderConfig) : { token: '' };
    this.token = config.token || '';
  }
  
  // Implement required methods
  validateConfig(config: any): ProviderConfig {
    // Validate and return the configuration
  }
  
  async checkAuth(integration: Integration): Promise<boolean> {
    // Check if authentication is valid
  }
  
  // Add provider-specific methods for fetching data
}
```

### 2. Register in Factory

Update `server/src/services/integrations/index.ts` to include the new provider:

```typescript
export * from './provider.service';

// In createIntegrationService function
export function createIntegrationService(integration: Integration): IntegrationService {
  if (!integration) {
    throw new Error('Integration is required');
  }
  
  switch (integration.type) {
    case 'GitHub':
      return new GitHubService(integration);
    case 'Provider': // Add your new provider here
      return new ProviderService(integration);
    // Add more cases for other integration types
    default:
      throw new Error(`Unsupported integration type: ${integration.type}`);
  }
}
```

### 3. Update Available Integration Types

Add your provider to the `AVAILABLE_INTEGRATION_TYPES` array in `server/src/routers/integrations.ts`:

```typescript
const AVAILABLE_INTEGRATION_TYPES: IntegrationTypeDetails[] = [
  // Existing integrations...
  {
    id: 'YourProvider',
    name: 'Your Provider',
    description: 'Connect to Your Provider to import resources',
    icon: 'icon-name',
    features: [
      { id: 'feature1', name: 'Feature 1', description: 'Import feature 1', enabled: true },
      { id: 'feature2', name: 'Feature 2', description: 'Import feature 2', enabled: true },
    ]
  },
];
```

### 4. Create API Endpoints

Add endpoints in `server/src/routers/integrations.ts`:

```typescript
// Initialize provider auth
initProviderAuth: protectedProcedure
  .input(z.object({
    workspaceId: z.string(),
    features: z.array(z.string()).optional(),
    redirectUrl: z.string().url(),
    providerType: IntegrationTypeSchema
  }))
  .mutation(async ({ input }) => {
    // Create a pending integration and return auth URL
  }),

// Handle provider OAuth callback
completeProviderAuth: protectedProcedure
  .input(z.object({
    code: z.string(),
    state: z.string(),
  }))
  .mutation(async ({ input }) => {
    // Exchange code for token and update integration
  }),
```

## Data Processing

Integrations pull data from external systems and store it in the `Source` table with appropriate metadata. The data processing flow typically includes:

1. **Fetching**: Retrieving data from external APIs
2. **Chunking**: Breaking content into searchable pieces
3. **Indexing**: Preparing content for search
4. **Summarizing**: Creating hierarchical summaries of content

### Source Structure

The integration system uses a hierarchical data model:

- `Source`: Top-level entity representing external content
- `SourceChunk`: Processed chunks of content for efficient retrieval
- `ChunkSummary`: Hierarchical summaries for better understanding

```typescript
model Source {
  id            String       @id @default(cuid())
  name          String
  type          SourceType
  workspaceId   String
  workspace     Workspace    @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  status        SourceStatus @default(Pending)
  content       String?      @db.Text
  // Other fields and relations...
  chunks        SourceChunk[]
  changedFiles  ChangedFile[]
}
```

## Change Monitoring

The integration system supports monitoring changes in external systems through:

1. **Webhooks**: Receiving real-time notifications from services that support them
2. **Polling**: Periodically checking for changes in services that don't support webhooks

The `Integration` model includes fields for tracking monitoring configuration:

```typescript
// Change Monitoring Configuration
monitorChanges     Boolean  @default(false)
webhookSecret      String?
pollingEnabled     Boolean  @default(false)
pollingInterval    Int?     // In minutes
lastPolledAt       DateTime?
nextPollAt         DateTime?
```

## Integration Synchronization

The system includes a background synchronization service that:

1. Periodically checks for integrations that need updating
2. Updates integration data based on configuration and last update time
3. Handles errors and retries

```typescript
// From your code
syncIntegration: protectedProcedure
  .input(z.object({ id: z.string() }))
  .mutation(async ({ input }) => {
    // Start the sync (non-blocking)
    integrationSyncService.syncIntegration(input.id)
      .catch(error => console.error(`Error syncing integration ${input.id}:`, error));
    // Return response
  }),
```

## Troubleshooting

### Common Issues

1. **Callback URL Mismatch**: Ensure that the redirect URI in your OAuth app configuration matches exactly what's in your environment variables.

2. **CORS Issues**: If you encounter CORS errors, verify that your server's CORS configuration allows requests from the client's origin.

3. **Token Expiration**: If integrations stop working after some time, check if the access token has expired and implement a refresh token flow.

### Debugging Tips

1. Check browser console for errors during the OAuth flow.
2. Verify that the environment variables are correctly set.
3. Test the OAuth flow manually using the provider's developer tools.
4. Check the network tab in developer tools to see if the callback requests are being made correctly.
5. Review Redis state for OAuth processes.

## Security Considerations

- Store all sensitive information (client secrets, tokens) on the server only
- Use HTTPS for all OAuth-related communication
- Validate state parameters to prevent CSRF attacks
- Implement proper error handling for failed authorization attempts
- Consider rate limiting to prevent abuse 

## Usage

To use integrations in your application:

1. **Setup**: Configure necessary environment variables for your integration providers
2. **Connect**: Use the provided OAuth flow to connect to external services
3. **Configure**: Select specific repositories, projects, or resources to sync
4. **Sync**: Trigger synchronization manually or wait for automatic sync
5. **Access**: Query synchronized data through the application's API

```typescript
// Example of manually triggering sync for all integrations
const syncAllIntegrations = async (workspaceId) => {
  try {
    await trpc.integrations.syncAllIntegrations.mutate({ workspaceId });
    // Handle success
  } catch (error) {
    // Handle error
  }
};
```

For specific integrations, use:

```typescript
const syncIntegration = async (integrationId) => {
  try {
    await trpc.integrations.syncIntegration.mutate({ id: integrationId });
    // Handle success
  } catch (error) {
    // Handle error
  }
};
```

Database operations should use the `prisma` client:

```typescript
import prisma from '@/lib/db';

// Get all active integrations for a workspace
const integrations = await prisma.integration.findMany({
  where: {
    workspaceId,
    status: 'active',
  },
});
``` 
