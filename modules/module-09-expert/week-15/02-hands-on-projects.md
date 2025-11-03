# Week 15: Hands-On Projects

This document provides detailed specifications and implementation guides for the three hands-on projects in Week 15.

## Table of Contents

1. [Project 1: Build a Custom Node for a Specific API](#project-1-build-a-custom-node-for-a-specific-api)
2. [Project 2: Create a Custom Trigger Node](#project-2-create-a-custom-trigger-node)
3. [Project 3: Develop a Node Package for Internal Use](#project-3-develop-a-node-package-for-internal-use)

---

## Project 1: Build a Custom Node for a Specific API

### Overview

Create a fully-featured custom node for an API service. This project teaches you the complete lifecycle of node development, from initial setup to publishing.

### Learning Outcomes

- Master regular node structure
- Implement multiple CRUD operations
- Handle API authentication
- Implement pagination
- Write comprehensive tests
- Create professional documentation

### Suggested APIs

Choose one of these APIs (or select your own):

1. **JSONPlaceholder** (Beginner-friendly)
   - Free fake REST API
   - URL: https://jsonplaceholder.typicode.com
   - No authentication required
   - Resources: users, posts, comments, albums, photos, todos

2. **GitHub API** (Intermediate)
   - Well-documented
   - OAuth2 and token auth
   - Resources: repos, issues, pull requests, users

3. **Notion API** (Advanced)
   - Complex data structures
   - OAuth2 authentication
   - Resources: pages, databases, blocks

4. **Your Company's Internal API** (Real-world)
   - Solve actual business needs
   - Custom requirements

### Project Structure

```
n8n-nodes-myapi/
├── credentials/
│   └── MyApiCredentials.ts
├── nodes/
│   └── MyApi/
│       ├── MyApi.node.ts
│       ├── myApi.svg
│       ├── descriptions/
│       │   ├── UserDescription.ts
│       │   ├── PostDescription.ts
│       │   └── index.ts
│       └── __tests__/
│           └── MyApi.node.spec.ts
├── package.json
├── tsconfig.json
├── jest.config.js
├── README.md
└── LICENSE
```

### Step-by-Step Implementation

#### Step 1: Project Setup (30 minutes)

```bash
# Use n8n starter template
git clone https://github.com/n8n-io/n8n-nodes-starter.git n8n-nodes-myapi
cd n8n-nodes-myapi

# Install dependencies
npm install

# Configure package.json
npm init
```

**package.json configuration:**

```json
{
  "name": "n8n-nodes-myapi",
  "version": "1.0.0",
  "description": "n8n node for MyAPI integration",
  "keywords": ["n8n-community-node-package"],
  "license": "MIT",
  "homepage": "https://github.com/yourusername/n8n-nodes-myapi",
  "author": {
    "name": "Your Name",
    "email": "your.email@example.com"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/yourusername/n8n-nodes-myapi.git"
  },
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc && npm run copy-icons",
    "copy-icons": "cp nodes/**/*.{svg,png} dist/nodes/",
    "dev": "tsc --watch",
    "format": "prettier --write .",
    "lint": "eslint . --ext .ts",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  },
  "files": ["dist"],
  "n8n": {
    "n8nNodesApiVersion": 1,
    "credentials": ["credentials/MyApiCredentials.ts"],
    "nodes": ["nodes/MyApi/MyApi.node.ts"]
  },
  "devDependencies": {
    "@types/jest": "^29.0.0",
    "@types/node": "^18.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "jest": "^29.0.0",
    "prettier": "^3.0.0",
    "ts-jest": "^29.0.0",
    "typescript": "^5.0.0"
  },
  "peerDependencies": {
    "n8n-workflow": "*"
  }
}
```

#### Step 2: Create Credentials (45 minutes)

**credentials/MyApiCredentials.ts:**

```typescript
import {
  IAuthenticateGeneric,
  ICredentialTestRequest,
  ICredentialType,
  INodeProperties,
} from 'n8n-workflow';

export class MyApiCredentials implements ICredentialType {
  name = 'myApiCredentials';
  displayName = 'MyAPI Credentials';
  documentationUrl = 'https://docs.myapi.com/authentication';

  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: {
        password: true,
      },
      default: '',
      required: true,
      description: 'Your MyAPI API key',
    },
    {
      displayName: 'Environment',
      name: 'environment',
      type: 'options',
      options: [
        {
          name: 'Production',
          value: 'production',
        },
        {
          name: 'Sandbox',
          value: 'sandbox',
        },
      ],
      default: 'production',
      description: 'The environment to use',
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

  test: ICredentialTestRequest = {
    request: {
      baseURL: '={{$credentials.environment === "production" ? "https://api.myapi.com" : "https://sandbox.myapi.com"}}',
      url: '/user',
    },
  };
}
```

#### Step 3: Build Node Structure (2 hours)

**nodes/MyApi/descriptions/UserDescription.ts:**

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
      {
        name: 'Get',
        value: 'get',
        description: 'Get a user by ID',
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
];

export const userFields: INodeProperties[] = [
  // USER ID (for get, update, delete)
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

  // NAME (for create, update)
  {
    displayName: 'Name',
    name: 'name',
    type: 'string',
    required: true,
    displayOptions: {
      show: {
        resource: ['user'],
        operation: ['create'],
      },
    },
    default: '',
    description: 'The name of the user',
  },

  // EMAIL (for create, update)
  {
    displayName: 'Email',
    name: 'email',
    type: 'string',
    placeholder: 'name@email.com',
    required: true,
    displayOptions: {
      show: {
        resource: ['user'],
        operation: ['create'],
      },
    },
    default: '',
    description: 'The email of the user',
  },

  // ADDITIONAL FIELDS (for create, update)
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
        displayName: 'Phone',
        name: 'phone',
        type: 'string',
        default: '',
        description: 'Phone number',
      },
      {
        displayName: 'Company',
        name: 'company',
        type: 'string',
        default: '',
        description: 'Company name',
      },
      {
        displayName: 'Role',
        name: 'role',
        type: 'options',
        options: [
          {
            name: 'Admin',
            value: 'admin',
          },
          {
            name: 'User',
            value: 'user',
          },
          {
            name: 'Guest',
            value: 'guest',
          },
        ],
        default: 'user',
        description: 'User role',
      },
    ],
  },

  // RETURN ALL
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

  // LIMIT
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
      maxValue: 500,
    },
    default: 50,
    description: 'Max number of results to return',
  },

  // FILTERS
  {
    displayName: 'Filters',
    name: 'filters',
    type: 'collection',
    placeholder: 'Add Filter',
    default: {},
    displayOptions: {
      show: {
        resource: ['user'],
        operation: ['getMany'],
      },
    },
    options: [
      {
        displayName: 'Role',
        name: 'role',
        type: 'options',
        options: [
          {
            name: 'Admin',
            value: 'admin',
          },
          {
            name: 'User',
            value: 'user',
          },
          {
            name: 'Guest',
            value: 'guest',
          },
        ],
        default: 'user',
        description: 'Filter by role',
      },
      {
        displayName: 'Created After',
        name: 'createdAfter',
        type: 'dateTime',
        default: '',
        description: 'Return users created after this date',
      },
    ],
  },
];
```

**nodes/MyApi/MyApi.node.ts:**

```typescript
import {
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
  IHttpRequestOptions,
  NodeOperationError,
} from 'n8n-workflow';

import { userOperations, userFields } from './descriptions/UserDescription';

export class MyApi implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'MyAPI',
    name: 'myApi',
    icon: 'file:myApi.svg',
    group: ['transform'],
    version: 1,
    subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
    description: 'Interact with MyAPI',
    defaults: {
      name: 'MyAPI',
    },
    inputs: ['main'],
    outputs: ['main'],
    credentials: [
      {
        name: 'myApiCredentials',
        required: true,
      },
    ],
    properties: [
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
        ],
        default: 'user',
      },
      ...userOperations,
      ...userFields,
    ],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];
    const resource = this.getNodeParameter('resource', 0);
    const operation = this.getNodeParameter('operation', 0);

    // Get credentials
    const credentials = await this.getCredentials('myApiCredentials');
    const baseUrl = credentials.environment === 'production'
      ? 'https://api.myapi.com'
      : 'https://sandbox.myapi.com';

    for (let i = 0; i < items.length; i++) {
      try {
        if (resource === 'user') {
          if (operation === 'create') {
            // CREATE USER
            const name = this.getNodeParameter('name', i) as string;
            const email = this.getNodeParameter('email', i) as string;
            const additionalFields = this.getNodeParameter(
              'additionalFields',
              i,
            ) as any;

            const body: any = {
              name,
              email,
              ...additionalFields,
            };

            const options: IHttpRequestOptions = {
              method: 'POST',
              url: `${baseUrl}/users`,
              body,
              json: true,
            };

            const response = await this.helpers.httpRequestWithAuthentication.call(
              this,
              'myApiCredentials',
              options,
            );

            returnData.push({
              json: response,
              pairedItem: { item: i },
            });

          } else if (operation === 'get') {
            // GET USER
            const userId = this.getNodeParameter('userId', i) as string;

            const options: IHttpRequestOptions = {
              method: 'GET',
              url: `${baseUrl}/users/${userId}`,
              json: true,
            };

            const response = await this.helpers.httpRequestWithAuthentication.call(
              this,
              'myApiCredentials',
              options,
            );

            returnData.push({
              json: response,
              pairedItem: { item: i },
            });

          } else if (operation === 'getMany') {
            // GET MANY USERS
            const returnAll = this.getNodeParameter('returnAll', i);
            const limit = this.getNodeParameter('limit', i, 50) as number;
            const filters = this.getNodeParameter('filters', i, {}) as any;

            let users: any[] = [];
            let page = 1;
            const perPage = 100;

            do {
              const qs: any = {
                page,
                per_page: perPage,
                ...filters,
              };

              const options: IHttpRequestOptions = {
                method: 'GET',
                url: `${baseUrl}/users`,
                qs,
                json: true,
              };

              const response = await this.helpers.httpRequestWithAuthentication.call(
                this,
                'myApiCredentials',
                options,
              );

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
            // UPDATE USER
            const userId = this.getNodeParameter('userId', i) as string;
            const additionalFields = this.getNodeParameter(
              'additionalFields',
              i,
            ) as any;

            if (Object.keys(additionalFields).length === 0) {
              throw new NodeOperationError(
                this.getNode(),
                'At least one field must be updated',
                { itemIndex: i },
              );
            }

            const options: IHttpRequestOptions = {
              method: 'PUT',
              url: `${baseUrl}/users/${userId}`,
              body: additionalFields,
              json: true,
            };

            const response = await this.helpers.httpRequestWithAuthentication.call(
              this,
              'myApiCredentials',
              options,
            );

            returnData.push({
              json: response,
              pairedItem: { item: i },
            });

          } else if (operation === 'delete') {
            // DELETE USER
            const userId = this.getNodeParameter('userId', i) as string;

            const options: IHttpRequestOptions = {
              method: 'DELETE',
              url: `${baseUrl}/users/${userId}`,
              json: true,
            };

            await this.helpers.httpRequestWithAuthentication.call(
              this,
              'myApiCredentials',
              options,
            );

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

#### Step 4: Write Tests (1.5 hours)

**nodes/MyApi/__tests__/MyApi.node.spec.ts:**

```typescript
import { IExecuteFunctions } from 'n8n-workflow';
import { MyApi } from '../MyApi.node';

describe('MyApi', () => {
  let myApi: MyApi;
  let mockExecuteFunctions: IExecuteFunctions;

  beforeEach(() => {
    myApi = new MyApi();

    mockExecuteFunctions = {
      getInputData: jest.fn(),
      getNodeParameter: jest.fn(),
      getCredentials: jest.fn(),
      getNode: jest.fn().mockReturnValue({ name: 'MyApi' }),
      helpers: {
        httpRequestWithAuthentication: jest.fn(),
      },
      continueOnFail: jest.fn().mockReturnValue(false),
    } as unknown as IExecuteFunctions;
  });

  describe('User Operations', () => {
    describe('create', () => {
      it('should create a user successfully', async () => {
        const mockUser = {
          id: '123',
          name: 'John Doe',
          email: 'john@example.com',
        };

        (mockExecuteFunctions.getInputData as jest.Mock).mockReturnValue([
          { json: {} },
        ]);

        (mockExecuteFunctions.getNodeParameter as jest.Mock)
          .mockReturnValueOnce('user')
          .mockReturnValueOnce('create')
          .mockReturnValueOnce('John Doe')
          .mockReturnValueOnce('john@example.com')
          .mockReturnValueOnce({});

        (mockExecuteFunctions.getCredentials as jest.Mock).mockResolvedValue({
          environment: 'production',
          apiKey: 'test-key',
        });

        (
          mockExecuteFunctions.helpers.httpRequestWithAuthentication as jest.Mock
        ).mockResolvedValue(mockUser);

        const result = await myApi.execute.call(mockExecuteFunctions);

        expect(result).toEqual([
          [
            {
              json: mockUser,
              pairedItem: { item: 0 },
            },
          ],
        ]);

        expect(
          mockExecuteFunctions.helpers.httpRequestWithAuthentication,
        ).toHaveBeenCalledWith('myApiCredentials', {
          method: 'POST',
          url: 'https://api.myapi.com/users',
          body: {
            name: 'John Doe',
            email: 'john@example.com',
          },
          json: true,
        });
      });
    });

    describe('get', () => {
      it('should get a user by ID', async () => {
        const userId = '123';
        const mockUser = {
          id: userId,
          name: 'John Doe',
          email: 'john@example.com',
        };

        (mockExecuteFunctions.getInputData as jest.Mock).mockReturnValue([
          { json: {} },
        ]);

        (mockExecuteFunctions.getNodeParameter as jest.Mock)
          .mockReturnValueOnce('user')
          .mockReturnValueOnce('get')
          .mockReturnValueOnce(userId);

        (mockExecuteFunctions.getCredentials as jest.Mock).mockResolvedValue({
          environment: 'production',
          apiKey: 'test-key',
        });

        (
          mockExecuteFunctions.helpers.httpRequestWithAuthentication as jest.Mock
        ).mockResolvedValue(mockUser);

        const result = await myApi.execute.call(mockExecuteFunctions);

        expect(result[0][0].json).toEqual(mockUser);
      });
    });

    describe('error handling', () => {
      it('should handle errors with continueOnFail', async () => {
        (mockExecuteFunctions.getInputData as jest.Mock).mockReturnValue([
          { json: {} },
        ]);

        (mockExecuteFunctions.getNodeParameter as jest.Mock)
          .mockReturnValueOnce('user')
          .mockReturnValueOnce('get')
          .mockReturnValueOnce('invalid-id');

        (mockExecuteFunctions.getCredentials as jest.Mock).mockResolvedValue({
          environment: 'production',
          apiKey: 'test-key',
        });

        (
          mockExecuteFunctions.helpers.httpRequestWithAuthentication as jest.Mock
        ).mockRejectedValue(new Error('User not found'));

        (mockExecuteFunctions.continueOnFail as jest.Mock).mockReturnValue(true);

        const result = await myApi.execute.call(mockExecuteFunctions);

        expect(result[0][0].json).toHaveProperty('error');
        expect(result[0][0].json.error).toBe('User not found');
      });
    });
  });
});
```

#### Step 5: Documentation (1 hour)

Create comprehensive README.md with:
- Installation instructions
- Available operations
- Usage examples
- Credential setup
- Troubleshooting

#### Step 6: Testing & Refinement (2 hours)

```bash
# Run tests
npm test

# Test coverage
npm test -- --coverage

# Build
npm run build

# Link locally
npm link

# Test in n8n
cd ~/.n8n
npm link n8n-nodes-myapi
n8n start
```

### Deliverables

- [ ] Working node with all CRUD operations
- [ ] Credential type with testing
- [ ] Unit tests with >80% coverage
- [ ] Complete README.md
- [ ] Icon (SVG)
- [ ] package.json properly configured
- [ ] Successfully tested in n8n

### Success Criteria

- All operations work correctly
- Error handling is robust
- Tests pass with good coverage
- Code follows n8n conventions
- Documentation is clear and complete
- Node installs and runs in n8n

---

## Project 2: Create a Custom Trigger Node

### Overview

Build a trigger node that listens for events from an external service using webhooks or polling.

### Learning Outcomes

- Understand trigger node architecture
- Implement webhook registration
- Handle activation/deactivation
- Manage trigger state
- Process real-time events

### Implementation Options

#### Option A: Webhook Trigger

**Use Case:** Service that supports webhooks (e.g., GitHub, Stripe, custom service)

**nodes/MyApiTrigger/MyApiTrigger.node.ts:**

```typescript
import {
  IHookFunctions,
  IWebhookFunctions,
  IWebhookResponseData,
  INodeType,
  INodeTypeDescription,
} from 'n8n-workflow';

export class MyApiTrigger implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'MyAPI Trigger',
    name: 'myApiTrigger',
    icon: 'file:myApi.svg',
    group: ['trigger'],
    version: 1,
    description: 'Starts the workflow when MyAPI events occur',
    defaults: {
      name: 'MyAPI Trigger',
    },
    inputs: [],
    outputs: ['main'],
    credentials: [
      {
        name: 'myApiCredentials',
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
            description: 'Triggered when a new user is created',
          },
          {
            name: 'User Updated',
            value: 'user.updated',
            description: 'Triggered when a user is updated',
          },
          {
            name: 'User Deleted',
            value: 'user.deleted',
            description: 'Triggered when a user is deleted',
          },
        ],
        default: ['user.created'],
        required: true,
        description: 'Events that trigger the workflow',
      },
    ],
  };

  // Webhook management methods
  webhookMethods = {
    default: {
      async checkExists(this: IHookFunctions): Promise<boolean> {
        const webhookData = this.getWorkflowStaticData('node');
        if (webhookData.webhookId === undefined) {
          return false;
        }

        const credentials = await this.getCredentials('myApiCredentials');
        const baseUrl =
          credentials.environment === 'production'
            ? 'https://api.myapi.com'
            : 'https://sandbox.myapi.com';

        try {
          await this.helpers.request({
            method: 'GET',
            url: `${baseUrl}/webhooks/${webhookData.webhookId}`,
            headers: {
              Authorization: `Bearer ${credentials.apiKey}`,
            },
          });
          return true;
        } catch (error) {
          return false;
        }
      },

      async create(this: IHookFunctions): Promise<boolean> {
        const webhookUrl = this.getNodeWebhookUrl('default');
        const events = this.getNodeParameter('events') as string[];
        const webhookData = this.getWorkflowStaticData('node');

        const credentials = await this.getCredentials('myApiCredentials');
        const baseUrl =
          credentials.environment === 'production'
            ? 'https://api.myapi.com'
            : 'https://sandbox.myapi.com';

        const response = await this.helpers.request({
          method: 'POST',
          url: `${baseUrl}/webhooks`,
          headers: {
            Authorization: `Bearer ${credentials.apiKey}`,
            'Content-Type': 'application/json',
          },
          body: {
            url: webhookUrl,
            events,
            active: true,
          },
          json: true,
        });

        webhookData.webhookId = response.id;

        return true;
      },

      async delete(this: IHookFunctions): Promise<boolean> {
        const webhookData = this.getWorkflowStaticData('node');

        if (webhookData.webhookId !== undefined) {
          const credentials = await this.getCredentials('myApiCredentials');
          const baseUrl =
            credentials.environment === 'production'
              ? 'https://api.myapi.com'
              : 'https://sandbox.myapi.com';

          try {
            await this.helpers.request({
              method: 'DELETE',
              url: `${baseUrl}/webhooks/${webhookData.webhookId}`,
              headers: {
                Authorization: `Bearer ${credentials.apiKey}`,
              },
            });
          } catch (error) {
            return false;
          }

          delete webhookData.webhookId;
        }

        return true;
      },
    },
  };

  async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const bodyData = this.getBodyData();
    const headerData = this.getHeaderData();

    // Verify webhook signature (if applicable)
    const signature = headerData['x-myapi-signature'] as string;
    // Add signature verification logic here

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

#### Option B: Polling Trigger

**Use Case:** Service that doesn't support webhooks

```typescript
import {
  ITriggerFunctions,
  ITriggerResponse,
  INodeType,
  INodeTypeDescription,
} from 'n8n-workflow';

export class MyApiPolling implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'MyAPI Polling',
    name: 'myApiPolling',
    icon: 'file:myApi.svg',
    group: ['trigger'],
    version: 1,
    description: 'Polls MyAPI for new events',
    defaults: {
      name: 'MyAPI Polling',
    },
    inputs: [],
    outputs: ['main'],
    credentials: [
      {
        name: 'myApiCredentials',
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
      {
        displayName: 'Event',
        name: 'event',
        type: 'options',
        options: [
          {
            name: 'Created',
            value: 'created',
          },
          {
            name: 'Updated',
            value: 'updated',
          },
        ],
        default: 'created',
      },
    ],
  };

  async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    const credentials = await this.getCredentials('myApiCredentials');
    const staticData = this.getWorkflowStaticData('node');
    const resource = this.getNodeParameter('resource') as string;
    const event = this.getNodeParameter('event') as string;

    const baseUrl =
      credentials.environment === 'production'
        ? 'https://api.myapi.com'
        : 'https://sandbox.myapi.com';

    // Initialize last check timestamp
    if (!staticData.lastCheck) {
      staticData.lastCheck = new Date().toISOString();
    }

    const pollFunction = async () => {
      try {
        const response = await this.helpers.request({
          method: 'GET',
          url: `${baseUrl}/${resource}`,
          qs: {
            event,
            since: staticData.lastCheck,
          },
          headers: {
            Authorization: `Bearer ${credentials.apiKey}`,
          },
          json: true,
        });

        if (response.data && response.data.length > 0) {
          // Update last check time
          staticData.lastCheck = new Date().toISOString();

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

    // Poll interval (from workflow settings, default 60 seconds)
    const pollInterval = (this.getNodeParameter('pollTimes.interval', 60) as number) * 1000;

    // Set up interval
    const intervalObj = setInterval(pollFunction, pollInterval);

    // Initial poll
    await pollFunction();

    // Cleanup function
    async function closeFunction() {
      clearInterval(intervalObj);
    }

    return {
      closeFunction,
    };
  }
}
```

### Testing Triggers

**Test webhook locally:**

```bash
# Start n8n
n8n start

# Activate workflow with trigger
# Use tools like ngrok for public URL

# Send test webhook
curl -X POST http://localhost:5678/webhook/test \
  -H "Content-Type: application/json" \
  -d '{"event": "user.created", "data": {"id": "123"}}'
```

### Deliverables

- [ ] Working trigger node (webhook or polling)
- [ ] Proper activation/deactivation
- [ ] State management
- [ ] Event filtering
- [ ] Error handling
- [ ] Tests
- [ ] Documentation

---

## Project 3: Develop a Node Package for Internal Use

### Overview

Create a package of related nodes for your organization's internal tools and systems.

### Learning Outcomes

- Organize multiple nodes
- Share credentials across nodes
- Create consistent design patterns
- Manage private npm packages
- Build internal documentation

### Project Structure

```
n8n-nodes-company/
├── credentials/
│   ├── CompanyApiCredentials.ts
│   └── CompanyDatabaseCredentials.ts
├── nodes/
│   ├── CompanyCRM/
│   │   └── CompanyCRM.node.ts
│   ├── CompanyAnalytics/
│   │   └── CompanyAnalytics.node.ts
│   └── CompanyHR/
│       └── CompanyHR.node.ts
├── utils/
│   ├── api.ts
│   └── constants.ts
├── package.json
├── README.md
└── INTERNAL_DOCS.md
```

### Shared Utilities

**utils/api.ts:**

```typescript
import { IExecuteFunctions, IHttpRequestOptions } from 'n8n-workflow';

export async function companyApiRequest(
  this: IExecuteFunctions,
  method: string,
  endpoint: string,
  body?: any,
  qs?: any,
): Promise<any> {
  const credentials = await this.getCredentials('companyApiCredentials');

  const options: IHttpRequestOptions = {
    method,
    url: `${credentials.baseUrl}${endpoint}`,
    headers: {
      'Authorization': `Bearer ${credentials.apiKey}`,
      'Content-Type': 'application/json',
    },
    body,
    qs,
    json: true,
  };

  try {
    return await this.helpers.httpRequestWithAuthentication.call(
      this,
      'companyApiCredentials',
      options,
    );
  } catch (error) {
    throw new Error(`Company API Error: ${error.message}`);
  }
}
```

**utils/constants.ts:**

```typescript
export const API_VERSION = 'v2';

export const RESOURCES = {
  CUSTOMER: 'customer',
  ORDER: 'order',
  PRODUCT: 'product',
} as const;

export const OPERATIONS = {
  CREATE: 'create',
  GET: 'get',
  GET_MANY: 'getMany',
  UPDATE: 'update',
  DELETE: 'delete',
} as const;
```

### Private npm Package

**package.json:**

```json
{
  "name": "@your-company/n8n-nodes",
  "version": "1.0.0",
  "description": "Internal n8n nodes for Company systems",
  "private": true,
  "keywords": ["n8n-community-node-package"],
  "main": "dist/index.js",
  "n8n": {
    "n8nNodesApiVersion": 1,
    "credentials": [
      "credentials/CompanyApiCredentials.ts",
      "credentials/CompanyDatabaseCredentials.ts"
    ],
    "nodes": [
      "nodes/CompanyCRM/CompanyCRM.node.ts",
      "nodes/CompanyAnalytics/CompanyAnalytics.node.ts",
      "nodes/CompanyHR/CompanyHR.node.ts"
    ]
  }
}
```

**Publishing to private registry:**

```bash
# Set registry
npm config set registry https://npm.your-company.com

# Login
npm login

# Publish
npm publish
```

### Internal Documentation

**INTERNAL_DOCS.md:**

```markdown
# Company n8n Nodes - Internal Documentation

## Installation

```bash
npm install @your-company/n8n-nodes
```

## Available Nodes

### Company CRM
- Customer management
- Deal tracking
- Contact synchronization

### Company Analytics
- Report generation
- Data export
- Metrics tracking

### Company HR
- Employee onboarding
- Leave management
- Performance reviews

## Credentials Setup

1. Generate API key from Company Admin Portal
2. Add credentials in n8n
3. Test connection

## Common Use Cases

### Use Case 1: Customer Onboarding
[Workflow example]

### Use Case 2: Monthly Reports
[Workflow example]

## Support

For issues or questions, contact: devops@your-company.com
```

### Deliverables

- [ ] At least 3 related nodes
- [ ] Shared credential types
- [ ] Shared utility functions
- [ ] Consistent design patterns
- [ ] Private npm package
- [ ] Internal documentation
- [ ] Example workflows
- [ ] Support process

---

## General Tips

### Development Workflow

1. **Plan**: Design node structure first
2. **Build**: Start with basic functionality
3. **Test**: Write tests as you go
4. **Refine**: Improve based on testing
5. **Document**: Write clear docs
6. **Deploy**: Publish or distribute

### Testing Strategy

- Unit tests for individual operations
- Integration tests for full workflows
- Manual testing in n8n UI
- Error scenario testing
- Edge case testing

### Code Quality

- Follow n8n conventions
- Use TypeScript strictly
- Write clear comments
- Keep functions small
- Handle errors gracefully

### Documentation

- Clear README
- Code comments
- Usage examples
- Troubleshooting guide
- API reference

---

## Assessment

Your projects will be evaluated on:

- **Functionality** (40%): Does it work correctly?
- **Code Quality** (20%): Is it well-written and maintainable?
- **Testing** (20%): Are tests comprehensive?
- **Documentation** (10%): Is it well-documented?
- **User Experience** (10%): Is it easy to use?

## Submission

Submit your completed projects with:

1. Source code repository
2. README with setup instructions
3. Test results and coverage
4. Screenshots or video demo
5. npm package (if published)

---

**Good luck with your projects! Remember, the goal is to learn through practice. Don't hesitate to experiment and iterate on your designs.**
