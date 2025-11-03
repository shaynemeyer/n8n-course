# Custom Node Development Guide

This comprehensive guide walks you through the complete process of creating custom n8n nodes from scratch.

## Table of Contents

1. [Environment Setup](#environment-setup)
2. [Understanding Node Architecture](#understanding-node-architecture)
3. [Creating Your First Node](#creating-your-first-node)
4. [Building a Complete Node](#building-a-complete-node)
5. [Creating Trigger Nodes](#creating-trigger-nodes)
6. [Implementing Credentials](#implementing-credentials)
7. [Testing Custom Nodes](#testing-custom-nodes)
8. [Publishing Your Node](#publishing-your-node)

---

## Environment Setup

### Prerequisites

Before starting, ensure you have:

```bash
node --version  # v18.x or later
npm --version   # v9.x or later
```

### Option 1: Using n8n-nodes-starter (Recommended)

The fastest way to get started:

```bash
# Clone the starter template
git clone https://github.com/n8n-io/n8n-nodes-starter.git my-custom-node
cd my-custom-node

# Install dependencies
npm install

# Build the node
npm run build

# Link for local testing
npm link
```

### Option 2: Manual Setup

Create a new project from scratch:

```bash
# Create project directory
mkdir n8n-nodes-mynode
cd n8n-nodes-mynode

# Initialize npm project
npm init -y

# Install dependencies
npm install n8n-workflow

# Install dev dependencies
npm install --save-dev \
  typescript \
  @types/node \
  ts-node \
  @typescript-eslint/eslint-plugin \
  @typescript-eslint/parser \
  eslint \
  prettier
```

### Project Structure

```
n8n-nodes-mynode/
├── credentials/
│   └── MyServiceApi.credentials.ts
├── nodes/
│   ├── MyService/
│   │   ├── MyService.node.ts
│   │   ├── myService.svg
│   │   └── descriptions/
│   │       ├── UserDescription.ts
│   │       └── PostDescription.ts
├── package.json
├── tsconfig.json
├── .eslintrc.js
└── README.md
```

### TypeScript Configuration

Create `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2019",
    "module": "commonjs",
    "lib": ["ES2019"],
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./",
    "removeComments": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "node",
    "resolveJsonModule": true
  },
  "include": ["nodes/**/*", "credentials/**/*"],
  "exclude": ["node_modules", "**/*.spec.ts"]
}
```

### Package.json Configuration

```json
{
  "name": "n8n-nodes-myservice",
  "version": "1.0.0",
  "description": "n8n node for MyService API",
  "keywords": ["n8n-community-node-package"],
  "license": "MIT",
  "homepage": "https://github.com/username/n8n-nodes-myservice",
  "author": {
    "name": "Your Name",
    "email": "your.email@example.com"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/username/n8n-nodes-myservice.git"
  },
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc && npm run copy-files",
    "copy-files": "cp nodes/**/*.{png,svg} dist/nodes/",
    "dev": "tsc --watch",
    "format": "prettier --write .",
    "lint": "eslint . --ext .ts",
    "test": "jest"
  },
  "files": ["dist"],
  "n8n": {
    "n8nNodesApiVersion": 1,
    "credentials": ["credentials/MyServiceApi.credentials.ts"],
    "nodes": ["nodes/MyService/MyService.node.ts"]
  },
  "devDependencies": {
    "@types/node": "^18.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "typescript": "^5.0.0"
  },
  "peerDependencies": {
    "n8n-workflow": "*"
  }
}
```

---

## Understanding Node Architecture

### Node Lifecycle

1. **Registration**: Node is loaded into n8n
2. **Instantiation**: Node instance created when added to workflow
3. **Parameter Collection**: Parameters gathered from UI
4. **Credential Loading**: Credentials decrypted and loaded
5. **Execution**: Execute method called for each workflow run
6. **Cleanup**: Resources released after execution

### Core Interfaces

```typescript
import {
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
} from 'n8n-workflow';
```

**Key Types:**

- `INodeType`: Main node interface
- `INodeTypeDescription`: Node metadata and parameters
- `IExecuteFunctions`: Execution context and helpers
- `INodeExecutionData`: Input/output data structure
- `INodePropertyOptions`: Parameter options
- `ICredentialType`: Credential definition

### Node Properties

```typescript
interface INodeTypeDescription {
  displayName: string;          // UI display name
  name: string;                  // Internal identifier (camelCase)
  icon: string;                  // Icon path
  group: string[];               // Node category
  version: number | number[];    // Node version
  subtitle?: string;             // Dynamic subtitle
  description: string;           // Node description
  defaults: {
    name: string;                // Default instance name
    color?: string;              // Node color
  };
  inputs: string[];              // Input types
  outputs: string[];             // Output types
  credentials?: INodeCredentialDescription[];
  properties: INodeProperties[]; // Parameters
}
```

### Data Structure

Every node processes arrays of items:

```typescript
interface INodeExecutionData {
  json: {                        // Main data
    [key: string]: any;
  };
  binary?: {                     // Binary data (files)
    [key: string]: IBinaryData;
  };
  pairedItem?: {                 // Links to input items
    item: number;
  };
}
```

---

## Creating Your First Node

### Minimal Node Example

Create `nodes/HelloWorld/HelloWorld.node.ts`:

```typescript
import {
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
} from 'n8n-workflow';

export class HelloWorld implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Hello World',
    name: 'helloWorld',
    icon: 'file:helloWorld.svg',
    group: ['transform'],
    version: 1,
    description: 'Returns a greeting message',
    defaults: {
      name: 'Hello World',
    },
    inputs: ['main'],
    outputs: ['main'],
    properties: [
      {
        displayName: 'Name',
        name: 'name',
        type: 'string',
        default: 'World',
        placeholder: 'Enter a name',
        description: 'The name to greet',
      },
    ],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
      const name = this.getNodeParameter('name', i) as string;

      returnData.push({
        json: {
          greeting: `Hello, ${name}!`,
          timestamp: new Date().toISOString(),
        },
        pairedItem: { item: i },
      });
    }

    return [returnData];
  }
}
```

### Testing Locally

```bash
# Build the node
npm run build

# Link globally
npm link

# In your n8n installation directory
npm link n8n-nodes-myservice

# Start n8n
n8n start
```

---

## Building a Complete Node

### API Integration Node

Let's build a complete node for a hypothetical API:

```typescript
import {
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
  IHttpRequestOptions,
} from 'n8n-workflow';

export class MyService implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'MyService',
    name: 'myService',
    icon: 'file:myService.svg',
    group: ['transform'],
    version: 1,
    subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
    description: 'Interact with MyService API',
    defaults: {
      name: 'MyService',
    },
    inputs: ['main'],
    outputs: ['main'],
    credentials: [
      {
        name: 'myServiceApi',
        required: true,
      },
    ],
    properties: [
      // Resource selector
      {
        displayName: 'Resource',
        name: 'resource',
        type: 'options',
        noDataExpression: true,
        options: [
          {
            name: 'User',
            value: 'user',
          },
          {
            name: 'Post',
            value: 'post',
          },
        ],
        default: 'user',
      },

      // User operations
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        displayOptions: {
          show: {
            resource: ['user'],
          },
        },
        options: [
          {
            name: 'Create',
            value: 'create',
            description: 'Create a new user',
            action: 'Create a user',
          },
          {
            name: 'Get',
            value: 'get',
            description: 'Get a user',
            action: 'Get a user',
          },
          {
            name: 'Get Many',
            value: 'getMany',
            description: 'Get many users',
            action: 'Get many users',
          },
          {
            name: 'Update',
            value: 'update',
            description: 'Update a user',
            action: 'Update a user',
          },
          {
            name: 'Delete',
            value: 'delete',
            description: 'Delete a user',
            action: 'Delete a user',
          },
        ],
        default: 'get',
      },

      // User ID (for get, update, delete)
      {
        displayName: 'User ID',
        name: 'userId',
        type: 'string',
        required: true,
        displayOptions: {
          show: {
            resource: ['user'],
            operation: ['get', 'update', 'delete'],
          },
        },
        default: '',
        description: 'The ID of the user',
      },

      // Additional fields for create/update
      {
        displayName: 'Additional Fields',
        name: 'additionalFields',
        type: 'collection',
        placeholder: 'Add Field',
        default: {},
        displayOptions: {
          show: {
            resource: ['user'],
            operation: ['create', 'update'],
          },
        },
        options: [
          {
            displayName: 'Email',
            name: 'email',
            type: 'string',
            default: '',
            placeholder: 'user@example.com',
          },
          {
            displayName: 'Name',
            name: 'name',
            type: 'string',
            default: '',
          },
          {
            displayName: 'Status',
            name: 'status',
            type: 'options',
            options: [
              {
                name: 'Active',
                value: 'active',
              },
              {
                name: 'Inactive',
                value: 'inactive',
              },
            ],
            default: 'active',
          },
        ],
      },

      // Pagination for getMany
      {
        displayName: 'Return All',
        name: 'returnAll',
        type: 'boolean',
        displayOptions: {
          show: {
            resource: ['user'],
            operation: ['getMany'],
          },
        },
        default: false,
        description: 'Whether to return all results or only up to a given limit',
      },
      {
        displayName: 'Limit',
        name: 'limit',
        type: 'number',
        displayOptions: {
          show: {
            resource: ['user'],
            operation: ['getMany'],
            returnAll: [false],
          },
        },
        typeOptions: {
          minValue: 1,
          maxValue: 100,
        },
        default: 50,
        description: 'Max number of results to return',
      },
    ],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];
    const resource = this.getNodeParameter('resource', 0);
    const operation = this.getNodeParameter('operation', 0);

    // Get credentials
    const credentials = await this.getCredentials('myServiceApi');

    for (let i = 0; i < items.length; i++) {
      try {
        if (resource === 'user') {
          if (operation === 'create') {
            // Create user
            const additionalFields = this.getNodeParameter('additionalFields', i);

            const body: any = {
              ...additionalFields,
            };

            const options: IHttpRequestOptions = {
              method: 'POST',
              url: `${credentials.domain}/api/users`,
              headers: {
                'Authorization': `Bearer ${credentials.apiKey}`,
                'Content-Type': 'application/json',
              },
              body,
              json: true,
            };

            const response = await this.helpers.httpRequest(options);

            returnData.push({
              json: response,
              pairedItem: { item: i },
            });

          } else if (operation === 'get') {
            // Get user
            const userId = this.getNodeParameter('userId', i) as string;

            const options: IHttpRequestOptions = {
              method: 'GET',
              url: `${credentials.domain}/api/users/${userId}`,
              headers: {
                'Authorization': `Bearer ${credentials.apiKey}`,
              },
              json: true,
            };

            const response = await this.helpers.httpRequest(options);

            returnData.push({
              json: response,
              pairedItem: { item: i },
            });

          } else if (operation === 'getMany') {
            // Get many users
            const returnAll = this.getNodeParameter('returnAll', i);
            const limit = this.getNodeParameter('limit', i, 50) as number;

            let users: any[] = [];
            let page = 1;
            const perPage = 100;

            do {
              const options: IHttpRequestOptions = {
                method: 'GET',
                url: `${credentials.domain}/api/users`,
                qs: {
                  page,
                  per_page: perPage,
                },
                headers: {
                  'Authorization': `Bearer ${credentials.apiKey}`,
                },
                json: true,
              };

              const response = await this.helpers.httpRequest(options);
              users = users.concat(response.data);

              if (!returnAll && users.length >= limit) {
                users = users.slice(0, limit);
                break;
              }

              if (!response.has_more) {
                break;
              }

              page++;
            } while (true);

            users.forEach((user) => {
              returnData.push({
                json: user,
                pairedItem: { item: i },
              });
            });

          } else if (operation === 'update') {
            // Update user
            const userId = this.getNodeParameter('userId', i) as string;
            const additionalFields = this.getNodeParameter('additionalFields', i);

            const options: IHttpRequestOptions = {
              method: 'PUT',
              url: `${credentials.domain}/api/users/${userId}`,
              headers: {
                'Authorization': `Bearer ${credentials.apiKey}`,
                'Content-Type': 'application/json',
              },
              body: additionalFields,
              json: true,
            };

            const response = await this.helpers.httpRequest(options);

            returnData.push({
              json: response,
              pairedItem: { item: i },
            });

          } else if (operation === 'delete') {
            // Delete user
            const userId = this.getNodeParameter('userId', i) as string;

            const options: IHttpRequestOptions = {
              method: 'DELETE',
              url: `${credentials.domain}/api/users/${userId}`,
              headers: {
                'Authorization': `Bearer ${credentials.apiKey}`,
              },
              json: true,
            };

            await this.helpers.httpRequest(options);

            returnData.push({
              json: {
                success: true,
                userId,
              },
              pairedItem: { item: i },
            });
          }
        }
      } catch (error) {
        if (this.continueOnFail()) {
          returnData.push({
            json: {
              error: error.message,
            },
            pairedItem: { item: i },
          });
          continue;
        }
        throw error;
      }
    }

    return [returnData];
  }
}
```

### Organizing Large Nodes

For complex nodes, split descriptions into separate files:

**nodes/MyService/descriptions/UserDescription.ts:**

```typescript
import { INodeProperties } from 'n8n-workflow';

export const userOperations: INodeProperties[] = [
  {
    displayName: 'Operation',
    name: 'operation',
    type: 'options',
    noDataExpression: true,
    displayOptions: {
      show: {
        resource: ['user'],
      },
    },
    options: [
      {
        name: 'Create',
        value: 'create',
        description: 'Create a new user',
        action: 'Create a user',
      },
      // ... more operations
    ],
    default: 'get',
  },
];

export const userFields: INodeProperties[] = [
  {
    displayName: 'User ID',
    name: 'userId',
    type: 'string',
    required: true,
    displayOptions: {
      show: {
        resource: ['user'],
        operation: ['get', 'update', 'delete'],
      },
    },
    default: '',
    description: 'The ID of the user',
  },
  // ... more fields
];
```

Import and use:

```typescript
import { userOperations, userFields } from './descriptions/UserDescription';

export class MyService implements INodeType {
  description: INodeTypeDescription = {
    // ... other properties
    properties: [
      resourceSelect,
      ...userOperations,
      ...userFields,
      ...postOperations,
      ...postFields,
    ],
  };
}
```

---

## Creating Trigger Nodes

### Webhook Trigger

```typescript
import {
  IWebhookFunctions,
  IWebhookResponseData,
  INodeType,
  INodeTypeDescription,
} from 'n8n-workflow';

export class MyServiceTrigger implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'MyService Trigger',
    name: 'myServiceTrigger',
    icon: 'file:myService.svg',
    group: ['trigger'],
    version: 1,
    description: 'Triggers on MyService webhook events',
    defaults: {
      name: 'MyService Trigger',
    },
    inputs: [],
    outputs: ['main'],
    credentials: [
      {
        name: 'myServiceApi',
        required: true,
      },
    ],
    webhooks: [
      {
        name: 'default',
        httpMethod: 'POST',
        responseMode: 'onReceived',
        path: 'webhook',
      },
    ],
    properties: [
      {
        displayName: 'Events',
        name: 'events',
        type: 'multiOptions',
        options: [
          {
            name: 'User Created',
            value: 'user.created',
          },
          {
            name: 'User Updated',
            value: 'user.updated',
          },
          {
            name: 'User Deleted',
            value: 'user.deleted',
          },
        ],
        default: ['user.created'],
        required: true,
        description: 'Events that trigger the workflow',
      },
    ],
  };

  // Called when workflow is activated
  async webhookMethods(this: IWebhookFunctions): Promise<any> {
    const webhookData = this.getWorkflowStaticData('node');
    const credentials = await this.getCredentials('myServiceApi');
    const webhookUrl = this.getNodeWebhookUrl('default');
    const events = this.getNodeParameter('events') as string[];

    if (this.getMode() === 'manual') {
      // Manual mode - return webhook URL for testing
      return {
        webhookUrl,
      };
    }

    // Register webhook with service
    const options = {
      method: 'POST',
      url: `${credentials.domain}/api/webhooks`,
      headers: {
        'Authorization': `Bearer ${credentials.apiKey}`,
      },
      body: {
        url: webhookUrl,
        events,
      },
      json: true,
    };

    const response = await this.helpers.request(options);

    // Store webhook ID for cleanup
    webhookData.webhookId = response.id;

    return true;
  }

  // Handle incoming webhook
  async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const bodyData = this.getBodyData();
    const headerData = this.getHeaderData();

    // Verify webhook signature
    const signature = headerData['x-myservice-signature'] as string;
    const credentials = await this.getCredentials('myServiceApi');

    // Validate signature here
    // ... signature validation logic

    return {
      workflowData: [
        [
          {
            json: bodyData,
          },
        ],
      ],
    };
  }
}
```

### Polling Trigger

```typescript
import {
  ITriggerFunctions,
  ITriggerResponse,
  INodeType,
  INodeTypeDescription,
} from 'n8n-workflow';

export class MyServicePolling implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'MyService Polling',
    name: 'myServicePolling',
    icon: 'file:myService.svg',
    group: ['trigger'],
    version: 1,
    description: 'Polls MyService for new items',
    defaults: {
      name: 'MyService Polling',
    },
    inputs: [],
    outputs: ['main'],
    credentials: [
      {
        name: 'myServiceApi',
        required: true,
      },
    ],
    polling: true,
    properties: [
      {
        displayName: 'Resource',
        name: 'resource',
        type: 'options',
        options: [
          {
            name: 'User',
            value: 'user',
          },
        ],
        default: 'user',
      },
    ],
  };

  async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    const credentials = await this.getCredentials('myServiceApi');
    const staticData = this.getWorkflowStaticData('node');
    const resource = this.getNodeParameter('resource') as string;

    // Store last poll timestamp
    if (!staticData.lastPoll) {
      staticData.lastPoll = new Date().toISOString();
    }

    const pollFunction = async () => {
      try {
        const options = {
          method: 'GET',
          url: `${credentials.domain}/api/${resource}`,
          qs: {
            since: staticData.lastPoll,
          },
          headers: {
            'Authorization': `Bearer ${credentials.apiKey}`,
          },
          json: true,
        };

        const response = await this.helpers.request(options);

        if (response.data && response.data.length > 0) {
          // Update last poll time
          staticData.lastPoll = new Date().toISOString();

          // Emit workflow for each new item
          response.data.forEach((item: any) => {
            this.emit([
              [
                {
                  json: item,
                },
              ],
            ]);
          });
        }
      } catch (error) {
        console.error('Polling error:', error);
      }
    };

    // Get polling interval from workflow settings (in seconds)
    const pollInterval = this.getNodeParameter('pollTimes.interval', 60) as number;

    // Set up interval
    const intervalObj = setInterval(pollFunction, pollInterval * 1000);

    // Initial poll
    await pollFunction();

    // Return cleanup function
    async function closeFunction() {
      clearInterval(intervalObj);
    }

    return {
      closeFunction,
    };
  }
}
```

---

## Implementing Credentials

### Basic API Key Credential

Create `credentials/MyServiceApi.credentials.ts`:

```typescript
import {
  IAuthenticateGeneric,
  ICredentialType,
  INodeProperties,
} from 'n8n-workflow';

export class MyServiceApi implements ICredentialType {
  name = 'myServiceApi';
  displayName = 'MyService API';
  documentationUrl = 'https://docs.myservice.com/api';
  properties: INodeProperties[] = [
    {
      displayName: 'Domain',
      name: 'domain',
      type: 'string',
      default: 'https://api.myservice.com',
      placeholder: 'https://api.myservice.com',
    },
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: {
        password: true,
      },
      default: '',
    },
  ];

  authenticate: IAuthenticateGeneric = {
    type: 'generic',
    properties: {
      headers: {
        'Authorization': '=Bearer {{$credentials.apiKey}}',
      },
    },
  };

  // Test credential
  async test(this: ICredentialTestFunctions): Promise<INodeCredentialTestResult> {
    const credentials = this.getCredentials();

    try {
      const options = {
        method: 'GET',
        url: `${credentials.domain}/api/user`,
        headers: {
          'Authorization': `Bearer ${credentials.apiKey}`,
        },
      };

      await this.helpers.request(options);

      return {
        status: 'OK',
        message: 'Authentication successful',
      };
    } catch (error) {
      return {
        status: 'Error',
        message: error.message,
      };
    }
  }
}
```

### OAuth2 Credential

```typescript
import {
  ICredentialType,
  INodeProperties,
} from 'n8n-workflow';

export class MyServiceOAuth2Api implements ICredentialType {
  name = 'myServiceOAuth2Api';
  extends = ['oAuth2Api'];
  displayName = 'MyService OAuth2 API';
  documentationUrl = 'https://docs.myservice.com/oauth';
  properties: INodeProperties[] = [
    {
      displayName: 'Grant Type',
      name: 'grantType',
      type: 'hidden',
      default: 'authorizationCode',
    },
    {
      displayName: 'Authorization URL',
      name: 'authUrl',
      type: 'hidden',
      default: 'https://myservice.com/oauth/authorize',
    },
    {
      displayName: 'Access Token URL',
      name: 'accessTokenUrl',
      type: 'hidden',
      default: 'https://myservice.com/oauth/token',
    },
    {
      displayName: 'Scope',
      name: 'scope',
      type: 'hidden',
      default: 'read write',
    },
    {
      displayName: 'Auth URI Query Parameters',
      name: 'authQueryParameters',
      type: 'hidden',
      default: '',
    },
    {
      displayName: 'Authentication',
      name: 'authentication',
      type: 'hidden',
      default: 'body',
    },
  ];
}
```

---

## Testing Custom Nodes

### Unit Testing Setup

Install test dependencies:

```bash
npm install --save-dev \
  jest \
  @types/jest \
  ts-jest
```

Create `jest.config.js`:

```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/nodes'],
  testMatch: ['**/*.spec.ts'],
  collectCoverageFrom: [
    'nodes/**/*.ts',
    '!nodes/**/*.spec.ts',
  ],
};
```

### Writing Tests

Create `nodes/MyService/MyService.node.spec.ts`:

```typescript
import { IExecuteFunctions } from 'n8n-workflow';
import { MyService } from './MyService.node';

describe('MyService', () => {
  let myService: MyService;
  let mockExecuteFunctions: IExecuteFunctions;

  beforeEach(() => {
    myService = new MyService();

    mockExecuteFunctions = {
      getInputData: jest.fn(),
      getNodeParameter: jest.fn(),
      getCredentials: jest.fn(),
      helpers: {
        httpRequest: jest.fn(),
      },
      continueOnFail: jest.fn().mockReturnValue(false),
    } as unknown as IExecuteFunctions;
  });

  describe('User Operations', () => {
    it('should get a user by ID', async () => {
      // Arrange
      const userId = '12345';
      const mockUser = {
        id: userId,
        name: 'John Doe',
        email: 'john@example.com',
      };

      (mockExecuteFunctions.getInputData as jest.Mock).mockReturnValue([
        { json: {} },
      ]);

      (mockExecuteFunctions.getNodeParameter as jest.Mock)
        .mockReturnValueOnce('user')  // resource
        .mockReturnValueOnce('get')   // operation
        .mockReturnValueOnce(userId); // userId

      (mockExecuteFunctions.getCredentials as jest.Mock).mockResolvedValue({
        domain: 'https://api.myservice.com',
        apiKey: 'test-key',
      });

      (mockExecuteFunctions.helpers.httpRequest as jest.Mock).mockResolvedValue(
        mockUser,
      );

      // Act
      const result = await myService.execute.call(
        mockExecuteFunctions,
      );

      // Assert
      expect(result).toEqual([
        [
          {
            json: mockUser,
            pairedItem: { item: 0 },
          },
        ],
      ]);

      expect(mockExecuteFunctions.helpers.httpRequest).toHaveBeenCalledWith({
        method: 'GET',
        url: `https://api.myservice.com/api/users/${userId}`,
        headers: {
          'Authorization': 'Bearer test-key',
        },
        json: true,
      });
    });

    it('should handle errors gracefully', async () => {
      // Arrange
      (mockExecuteFunctions.getInputData as jest.Mock).mockReturnValue([
        { json: {} },
      ]);

      (mockExecuteFunctions.getNodeParameter as jest.Mock)
        .mockReturnValueOnce('user')
        .mockReturnValueOnce('get')
        .mockReturnValueOnce('invalid-id');

      (mockExecuteFunctions.getCredentials as jest.Mock).mockResolvedValue({
        domain: 'https://api.myservice.com',
        apiKey: 'test-key',
      });

      (mockExecuteFunctions.helpers.httpRequest as jest.Mock).mockRejectedValue(
        new Error('User not found'),
      );

      (mockExecuteFunctions.continueOnFail as jest.Mock).mockReturnValue(true);

      // Act
      const result = await myService.execute.call(
        mockExecuteFunctions,
      );

      // Assert
      expect(result[0][0].json).toHaveProperty('error');
      expect(result[0][0].json.error).toBe('User not found');
    });
  });
});
```

### Run Tests

```bash
npm test                    # Run all tests
npm test -- --coverage      # With coverage
npm test -- --watch         # Watch mode
```

---

## Publishing Your Node

### Pre-publication Checklist

- [ ] All functionality working
- [ ] Tests passing with good coverage
- [ ] Documentation complete
- [ ] README.md with examples
- [ ] LICENSE file added
- [ ] package.json properly configured
- [ ] Icon created (SVG, 60x60px)
- [ ] Version number set (start with 1.0.0)
- [ ] Git repository created
- [ ] .gitignore configured
- [ ] .npmignore configured (if needed)

### README.md Template

```markdown
# n8n-nodes-myservice

This is an n8n community node that lets you use MyService in your n8n workflows.

MyService is a [describe what the service does].

[Installation](#installation)
[Operations](#operations)
[Credentials](#credentials)
[Compatibility](#compatibility)
[Usage](#usage)
[Resources](#resources)

## Installation

Follow the [installation guide](https://docs.n8n.io/integrations/community-nodes/installation/) in the n8n community nodes documentation.

## Operations

- User
  - Create a user
  - Get a user
  - Get many users
  - Update a user
  - Delete a user

## Credentials

To use this node, you'll need a MyService account and API key.

1. Sign up at [MyService](https://myservice.com)
2. Generate an API key in your account settings
3. Enter the API key in n8n

## Compatibility

Tested against n8n version 1.0.0+.

## Usage

[Provide example workflows or usage scenarios]

## Resources

- [n8n community nodes documentation](https://docs.n8n.io/integrations/community-nodes/)
- [MyService API docs](https://docs.myservice.com)

## License

[MIT](LICENSE.md)
```

### Publishing to npm

```bash
# Make sure you're logged into npm
npm login

# Build your node
npm run build

# Publish
npm publish

# Or for scoped packages
npm publish --access public
```

### Version Updates

```bash
# Patch version (bug fixes)
npm version patch

# Minor version (new features)
npm version minor

# Major version (breaking changes)
npm version major

# Then publish
npm publish
```

### Submitting to n8n Community

1. Publish to npm first
2. Go to [n8n community nodes](https://www.npmjs.com/search?q=keywords:n8n-community-node-package)
3. Ensure your package appears with the `n8n-community-node-package` keyword
4. Share in the n8n community forum

---

## Best Practices Summary

1. **Follow Conventions**: Study existing n8n nodes
2. **Type Everything**: Use TypeScript fully
3. **Handle Errors**: Graceful failures
4. **Test Thoroughly**: High coverage
5. **Document Well**: Clear, helpful docs
6. **Version Carefully**: Semantic versioning
7. **Be Consistent**: UI patterns and naming
8. **Optimize**: Pagination, rate limiting
9. **Secure**: Never expose credentials
10. **Contribute**: Share with community

---

## Next Steps

1. Build your first custom node
2. Test it thoroughly
3. Get feedback from n8n community
4. Publish to npm
5. Build more nodes
6. Contribute to n8n core

Happy node building!
