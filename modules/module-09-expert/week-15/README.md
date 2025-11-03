# Week 15: Custom Nodes and Extensions

## Overview

This week dives deep into n8n's architecture and teaches you how to create custom nodes and extensions. You'll learn TypeScript, understand the n8n node structure, build your own nodes, test them, and potentially contribute to the n8n community. This is the gateway to becoming an n8n expert and extending the platform to meet any need.

## Learning Objectives

By the end of this week, you will be able to:

- Understand n8n's architecture and plugin system
- Set up a custom node development environment
- Create custom regular nodes and trigger nodes
- Implement node parameters and credentials
- Handle node execution logic and error handling
- Write tests for custom nodes
- Build versioned nodes with migrations
- Package and publish nodes to npm
- Contribute to the n8n community
- Debug custom nodes effectively
- Implement best practices for node development
- Create reusable node components

## Topics Covered

1. **n8n Architecture Overview**
   - Core system architecture
   - Node lifecycle and execution
   - How nodes interact with the core
   - Plugin system and loading mechanism
   - Workflow execution engine
   - Data flow and transformation
   - Event system and hooks

2. **Development Environment Setup**
   - Required tools and dependencies
   - TypeScript configuration
   - Node.js version requirements
   - Development workspace setup
   - n8n-nodes-base cloning
   - Local n8n instance for testing
   - Hot reload configuration
   - Debugging setup

3. **Node Structure and Components**
   - Node class structure
   - INodeType interface
   - Node properties and metadata
   - Description object anatomy
   - Input and output definitions
   - Parameter definitions
   - Display options and conditions
   - Resource localization

4. **Creating Regular Nodes**
   - Basic node template
   - Implementing execute method
   - Processing input items
   - Making HTTP requests
   - Handling responses
   - Error handling patterns
   - Multiple operations support
   - Resource and operation structure

5. **Creating Trigger Nodes**
   - Webhook triggers
   - Polling triggers
   - Manual triggers
   - Managing trigger state
   - Activation and deactivation
   - Handling concurrent executions
   - Real-time event processing

6. **Node Parameters**
   - Parameter types (string, number, boolean, etc.)
   - Options and multi-options
   - Collections and fixed collections
   - Resource mapping
   - Conditional display options
   - Default values and validation
   - Dynamic parameters
   - Custom parameter components

7. **Credentials Management**
   - Creating credential types
   - OAuth2 implementation
   - API key authentication
   - Basic authentication
   - Custom authentication methods
   - Credential testing
   - Secure credential storage
   - Credential dependencies

8. **TypeScript for n8n**
   - Essential TypeScript concepts
   - n8n type definitions
   - Interface usage
   - Generic types in nodes
   - Type safety best practices
   - Common type patterns
   - Error typing

9. **Testing Custom Nodes**
   - Unit testing setup
   - Testing node execution
   - Mocking HTTP requests
   - Credential testing
   - Integration tests
   - Test coverage
   - CI/CD for nodes

10. **Versioning and Migrations**
    - Node versioning strategy
    - Breaking vs non-breaking changes
    - Writing migrations
    - Backward compatibility
    - Deprecation patterns
    - Version selection logic

11. **Publishing and Distribution**
    - npm package structure
    - Package.json configuration
    - Publishing to npm
    - Community nodes submission
    - Private node packages
    - Version management
    - Documentation requirements

12. **Contributing to n8n**
    - Understanding contribution guidelines
    - Forking and pull requests
    - Code style and standards
    - Documentation standards
    - Review process
    - Community engagement

## Week Structure

### Day 1: Architecture & Environment Setup
- Study n8n architecture
- Install development dependencies
- Clone n8n-nodes-base
- Set up TypeScript environment
- Configure debugging tools
- Create first node skeleton

### Day 2: Building Regular Nodes
- Create basic node structure
- Implement execute method
- Add node parameters
- Handle HTTP requests
- Implement error handling
- Test node locally

### Day 3: Advanced Node Features
- Add multiple operations
- Implement resource mapping
- Create dynamic parameters
- Add conditional display logic
- Implement pagination
- Handle rate limiting

### Day 4: Trigger Nodes & Credentials
- Build webhook trigger
- Create polling trigger
- Implement credential type
- Add OAuth2 authentication
- Test credential flow
- Handle credential errors

### Day 5: Testing & Quality
- Write unit tests
- Create integration tests
- Set up test mocking
- Add test coverage
- Implement CI/CD
- Code review preparation

### Day 6: Versioning & Publishing
- Implement node versioning
- Write migration logic
- Prepare npm package
- Write documentation
- Publish to npm
- Test published package

### Day 7: Projects & Contribution
- Work on hands-on projects
- Refine custom nodes
- Prepare community contribution
- Create documentation
- Test in production scenarios
- Plan next nodes

## Hands-On Projects

### Project 1: Build a Custom Node for a Specific API

Create a fully-featured custom node for an API that doesn't have an existing n8n node.

**Requirements:**
- Multiple operations (create, read, update, delete)
- Resource-based structure
- Proper authentication
- Error handling
- Pagination support
- Comprehensive parameter definitions
- Full test coverage

**Suggested APIs:**
- Airtable (advanced features)
- Linear
- Notion (if not already complete)
- Custom company API
- Industry-specific service

**Deliverables:**
- Working node implementation
- Credential type
- Unit and integration tests
- Documentation
- npm package (optional)

### Project 2: Create a Custom Trigger Node

Build a trigger node that listens for events from an external service.

**Requirements:**
- Webhook trigger implementation
- Optional polling fallback
- Proper activation/deactivation
- Event filtering options
- State management
- Error handling and retries
- Test coverage

**Suggested Services:**
- Custom webhook service
- Database change trigger
- File system watcher
- Message queue listener
- Custom event source

**Deliverables:**
- Working trigger node
- Activation/deactivation logic
- Event handling
- Tests
- Documentation

### Project 3: Develop a Node Package for Internal Use

Create a package of custom nodes for your organization's internal tools.

**Requirements:**
- Multiple related nodes
- Shared credentials
- Consistent design patterns
- Private npm package
- Internal documentation
- Version management
- Team collaboration setup

**Deliverables:**
- Node package with 2-3 nodes
- Shared credential types
- Complete test suite
- Internal documentation
- Deployment guide
- Training materials

## Prerequisites

Before starting this week:
- Completed Modules 1-8
- Strong JavaScript knowledge
- Basic TypeScript understanding (or willingness to learn)
- Experience with Node.js and npm
- Familiarity with REST APIs
- Git and GitHub proficiency
- Understanding of testing concepts
- Node.js 18+ installed

## Key Concepts

### Node Anatomy

```typescript
import { INodeType, INodeTypeDescription } from 'n8n-workflow';

export class CustomNode implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Custom Node',
    name: 'customNode',
    icon: 'file:customNode.svg',
    group: ['transform'],
    version: 1,
    description: 'Custom node description',
    defaults: {
      name: 'Custom Node',
    },
    inputs: ['main'],
    outputs: ['main'],
    credentials: [],
    properties: [],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    // Node execution logic
  }
}
```

### Parameter Types

**Common Parameter Types:**
- `string`: Text input
- `number`: Numeric input
- `boolean`: Checkbox
- `options`: Dropdown selection
- `multiOptions`: Multi-select dropdown
- `collection`: Grouped parameters
- `fixedCollection`: Multiple instances of grouped parameters
- `json`: JSON editor
- `color`: Color picker
- `dateTime`: Date/time picker
- `hidden`: Hidden value
- `notice`: Information display

### Credential Structure

```typescript
import { ICredentialType, INodeProperties } from 'n8n-workflow';

export class CustomApi implements ICredentialType {
  name = 'customApi';
  displayName = 'Custom API';
  documentationUrl = 'https://docs.example.com';
  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      default: '',
      required: true,
      typeOptions: {
        password: true,
      },
    },
  ];

  async authenticate(
    credentials: ICredentialDataDecryptedObject,
    requestOptions: IHttpRequestOptions,
  ): Promise<IHttpRequestOptions> {
    requestOptions.headers = {
      ...requestOptions.headers,
      'Authorization': `Bearer ${credentials.apiKey}`,
    };
    return requestOptions;
  }
}
```

### Execution Context

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();
  const returnData: INodeExecutionData[] = [];

  // Get parameters
  const operation = this.getNodeParameter('operation', 0) as string;

  // Get credentials
  const credentials = await this.getCredentials('customApi');

  // Loop through items
  for (let i = 0; i < items.length; i++) {
    try {
      // Get item-specific parameters
      const resourceId = this.getNodeParameter('resourceId', i) as string;

      // Make request
      const response = await this.helpers.request({
        method: 'GET',
        url: `https://api.example.com/resource/${resourceId}`,
        headers: {
          'Authorization': `Bearer ${credentials.apiKey}`,
        },
      });

      // Add to return data
      returnData.push({
        json: response,
        pairedItem: { item: i },
      });
    } catch (error) {
      // Handle errors
      if (this.continueOnFail()) {
        returnData.push({
          json: { error: error.message },
          pairedItem: { item: i },
        });
        continue;
      }
      throw error;
    }
  }

  return [returnData];
}
```

## Development Tools

### Required Software

- **Node.js**: v18.x or later
- **npm**: 9.x or later (or pnpm)
- **TypeScript**: 5.x
- **Git**: For version control
- **Code Editor**: VS Code recommended
- **n8n CLI**: For testing

### Useful VS Code Extensions

- ESLint
- Prettier
- TypeScript Hero
- GitLens
- Thunder Client (API testing)
- Jest Runner

### Testing Tools

- Jest: Unit testing framework
- nock: HTTP request mocking
- @n8n/n8n-nodes-dev-utils: n8n testing utilities

## Best Practices

### Code Quality

1. **Follow n8n Conventions**: Consistent with existing nodes
2. **Use TypeScript**: Full type safety
3. **Error Handling**: Graceful failures
4. **Input Validation**: Validate all inputs
5. **Resource Management**: Clean up resources
6. **Performance**: Optimize for large datasets
7. **Documentation**: Clear parameter descriptions

### Node Design

1. **Single Responsibility**: One clear purpose
2. **Intuitive Parameters**: Easy to understand
3. **Sensible Defaults**: Reduce configuration
4. **Clear Error Messages**: Help users debug
5. **Consistent Naming**: Follow conventions
6. **Resource Organization**: Logical grouping
7. **Progressive Disclosure**: Hide complexity

### Testing Strategy

1. **Unit Tests**: Test individual functions
2. **Integration Tests**: Test full execution
3. **Error Cases**: Test failure scenarios
4. **Edge Cases**: Boundary conditions
5. **Credential Tests**: Authentication flows
6. **Mock External Calls**: Don't call real APIs
7. **Coverage Goals**: Aim for 80%+

### Version Management

1. **Semantic Versioning**: Follow semver
2. **Backward Compatibility**: When possible
3. **Migration Guides**: Clear upgrade paths
4. **Changelog**: Document all changes
5. **Deprecation Warnings**: Before removal
6. **Version Selection**: Let users choose

## Common Patterns

### Resource/Operation Structure

```typescript
{
  displayName: 'Resource',
  name: 'resource',
  type: 'options',
  options: [
    { name: 'User', value: 'user' },
    { name: 'Post', value: 'post' },
  ],
  default: 'user',
},
{
  displayName: 'Operation',
  name: 'operation',
  type: 'options',
  displayOptions: {
    show: { resource: ['user'] },
  },
  options: [
    { name: 'Create', value: 'create' },
    { name: 'Get', value: 'get' },
    { name: 'Update', value: 'update' },
    { name: 'Delete', value: 'delete' },
  ],
  default: 'get',
}
```

### Pagination Handling

```typescript
async function getAllResults(
  this: IExecuteFunctions,
  endpoint: string,
): Promise<any[]> {
  const returnAll = this.getNodeParameter('returnAll', 0) as boolean;
  const limit = this.getNodeParameter('limit', 0, 100) as number;

  let results: any[] = [];
  let page = 1;

  do {
    const response = await this.helpers.request({
      method: 'GET',
      url: `${endpoint}?page=${page}&per_page=100`,
    });

    results = results.concat(response.data);

    if (!returnAll && results.length >= limit) {
      return results.slice(0, limit);
    }

    if (!response.pagination?.has_more) {
      break;
    }

    page++;
  } while (true);

  return results;
}
```

### Dynamic Parameters

```typescript
async loadOptions(this: ILoadOptionsFunctions): Promise<INodePropertyOptions[]> {
  const credentials = await this.getCredentials('customApi');

  const response = await this.helpers.request({
    method: 'GET',
    url: 'https://api.example.com/options',
    headers: {
      'Authorization': `Bearer ${credentials.apiKey}`,
    },
  });

  return response.data.map((item: any) => ({
    name: item.name,
    value: item.id,
  }));
}
```

## Troubleshooting

### Common Issues

**Issue: Node doesn't appear in n8n**
- Check package.json node definition
- Verify node file naming
- Restart n8n completely
- Check for TypeScript compilation errors

**Issue: Credentials not working**
- Verify credential type name matches
- Check authentication implementation
- Test credential independently
- Review credential documentation

**Issue: Type errors**
- Update @types packages
- Check n8n-workflow version
- Verify interface implementations
- Use proper generic types

**Issue: Tests failing**
- Check mock data structure
- Verify async/await usage
- Review error handling
- Check test environment setup

## Security Considerations

### API Keys and Secrets

1. **Never Hardcode**: Use credentials system
2. **Secure Storage**: Encrypted at rest
3. **Transmission Security**: HTTPS only
4. **Rotation Support**: Allow key updates
5. **Minimal Permissions**: Least privilege
6. **Audit Logging**: Track credential usage

### Data Handling

1. **Sanitize Inputs**: Prevent injection
2. **Validate Outputs**: Check data structure
3. **Error Messages**: Don't leak secrets
4. **PII Handling**: Follow regulations
5. **Rate Limiting**: Respect API limits
6. **Resource Cleanup**: Close connections

## Publishing Checklist

Before publishing your node:

- [ ] Code follows n8n conventions
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Examples provided
- [ ] Error handling implemented
- [ ] Credentials tested
- [ ] Version number set
- [ ] Changelog updated
- [ ] README.md written
- [ ] package.json configured
- [ ] Icon created (SVG)
- [ ] License specified
- [ ] Repository URL set
- [ ] Keywords added
- [ ] Peer dependencies correct

## Assessment Criteria

You should be able to:
- Set up custom node development environment
- Create functional regular nodes
- Implement trigger nodes
- Build credential types
- Write comprehensive tests
- Package and publish nodes
- Follow n8n conventions
- Contribute to n8n community
- Debug node issues
- Implement best practices
- Create production-ready nodes

## Tips for Success

1. **Study Existing Nodes**: Learn from n8n-nodes-base
2. **Start Simple**: Basic functionality first
3. **Iterate**: Refine based on testing
4. **Document Early**: Write docs as you code
5. **Test Thoroughly**: Catch issues early
6. **Seek Feedback**: Community is helpful
7. **Version Carefully**: Plan for changes
8. **Stay Updated**: Follow n8n releases
9. **Contribute Back**: Help others learn
10. **Build Portfolio**: Showcase your work

## Common Pitfalls

- Overcomplicating node design
- Inadequate error handling
- Poor parameter organization
- Missing input validation
- Insufficient testing
- Not following conventions
- Hardcoding values
- Ignoring backward compatibility
- Incomplete documentation
- Not testing with real workflows

## Week Deliverables

By end of week, you should have:

- [ ] Development environment configured
- [ ] At least one custom regular node
- [ ] At least one custom trigger node
- [ ] Custom credential type
- [ ] Comprehensive test suite
- [ ] One complete hands-on project
- [ ] Published npm package (optional)
- [ ] Documentation complete
- [ ] Understanding of contribution process

## Resources

### Official Documentation

- [n8n Node Development Docs](https://docs.n8n.io/integrations/creating-nodes/)
- [n8n API Reference](https://docs.n8n.io/api/reference/)
- [Creating Custom Nodes](https://docs.n8n.io/integrations/creating-nodes/build/)
- [n8n-nodes-base Repository](https://github.com/n8n-io/n8n)
- [Community Nodes](https://docs.n8n.io/integrations/community-nodes/)

### Code Examples

- [n8n-nodes-base source](https://github.com/n8n-io/n8n/tree/master/packages/nodes-base/nodes)
- [n8n Node Starter](https://github.com/n8n-io/n8n-nodes-starter)
- [Example nodes](https://github.com/n8n-io/n8n-nodes-examples)

### TypeScript Resources

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)
- [TypeScript Deep Dive](https://basarat.gitbook.io/typescript/)
- [n8n Workflow Types](https://github.com/n8n-io/n8n/blob/master/packages/workflow/src/Interfaces.ts)

### Community

- [n8n Community Forum](https://community.n8n.io/)
- [n8n Discord](https://discord.gg/n8n)
- [GitHub Discussions](https://github.com/n8n-io/n8n/discussions)

## Next Steps

After completing this week:
1. Polish your custom nodes
2. Publish to npm if appropriate
3. Contribute to n8n community
4. Build node library for common needs
5. Move to Week 16: Capstone Project
6. Consider becoming a community contributor
7. Share your knowledge through tutorials

---

**Remember**: Custom node development is where you truly become an n8n expert. Take your time, study existing code, follow best practices, and don't hesitate to ask the community for help. Your custom nodes can help thousands of other n8n users!
