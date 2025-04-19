# Project Structure Overview

## Introduction

This document provides a comprehensive overview of the project structure, explaining the organization, component relationships, and file organization strategies. Understanding this structure is essential for all developers working on the project.

## High-Level Architecture

The project follows a microservices architecture with the following core components:

1. **API Core Service**: The main service that handles client requests, authentication, and route coordination
2. **Data Services**: Services focused on data processing, storage, and retrieval
3. **Integration Services**: Services that connect with external systems and APIs
4. **Supporting Services**: Infrastructure components for logging, monitoring, and administration

### Services Diagram

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  Client Apps    │────▶│  API Gateway    │────▶│  Auth Service   │
│                 │     │                 │     │                 │
└─────────────────┘     └────────┬────────┘     └─────────────────┘
                                 │
                                 ▼
         ┌───────────────────────────────────────────────┐
         │                                               │
         ▼                                               ▼
┌─────────────────┐                           ┌─────────────────┐
│                 │                           │                 │
│  Core Service   │                           │  Data Service   │
│                 │                           │                 │
└────────┬────────┘                           └────────┬────────┘
         │                                             │
         ▼                                             ▼
┌─────────────────┐                           ┌─────────────────┐
│                 │                           │                 │
│  Integration    │                           │  Supporting     │
│  Services       │                           │  Services       │
│                 │                           │                 │
└─────────────────┘                           └─────────────────┘
```

## Directory Structure

The project follows a modular structure to enable clear separation of concerns and ease of maintenance:

```
project-root/
├── api-core/                 # Main API service
│   ├── src/
│   │   ├── controllers/      # Request handlers
│   │   ├── middleware/       # Express middleware
│   │   ├── routes/           # API route definitions
│   │   ├── services/         # Business logic
│   │   ├── utils/            # Utility functions
│   │   └── app.js            # Express application
│   ├── tests/                # Tests for API service
│   ├── Dockerfile            # Container definition
│   └── package.json          # Dependencies
│
├── data-service/             # Data processing service
│   ├── src/
│   │   ├── models/           # Data models
│   │   ├── repositories/     # Data access layer
│   │   ├── services/         # Business logic
│   │   └── utils/            # Utility functions
│   ├── tests/                # Tests for data service
│   ├── Dockerfile            # Container definition
│   └── package.json          # Dependencies
│
├── auth-service/             # Authentication service
│   ├── src/
│   │   ├── controllers/      # Auth controllers
│   │   ├── models/           # User model
│   │   ├── services/         # Auth business logic
│   │   └── utils/            # Auth utilities
│   ├── tests/                # Tests for auth service
│   ├── Dockerfile            # Container definition
│   └── package.json          # Dependencies
│
├── integration-services/     # Integration services
│   ├── service-one/          # First integration service
│   ├── service-two/          # Second integration service
│   └── ...                   # Other integration services
│
├── frontend/                 # Frontend application
│   ├── src/
│   │   ├── components/       # UI components
│   │   ├── pages/            # Page components
│   │   ├── hooks/            # Custom hooks
│   │   ├── services/         # API clients
│   │   └── utils/            # Frontend utilities
│   ├── public/               # Static assets
│   ├── tests/                # Frontend tests
│   └── package.json          # Frontend dependencies
│
├── docs/                     # Documentation
│   ├── architecture/         # Architecture diagrams
│   ├── api/                  # API documentation
│   └── guides/               # Developer guides
│
├── scripts/                  # Utility scripts
│   ├── setup.sh              # Environment setup
│   ├── deploy.sh             # Deployment script
│   └── ci-cd/                # CI/CD scripts
│
├── infrastructure/           # Infrastructure as code
│   ├── docker/               # Docker configuration
│   │   └── docker-compose.yml # Service orchestration
│   ├── kubernetes/           # Kubernetes manifests
│   └── terraform/            # Infrastructure definitions
│
├── project/                  # Project documentation
│   ├── README.md             # Project overview
│   ├── project-requirements.md # Requirements
│   ├── project-plan.md       # Implementation plan
│   ├── test-implementation-plan.md # Testing plan
│   └── DEVLOG.md             # Development log
│
└── README.md                 # Root README
```

## Service Responsibility Breakdown

### API Core Service

**Primary Responsibility**: Handle incoming client requests, route to appropriate services, and return responses

**Key Components**:
1. **Routes**: Define API endpoints and validation
2. **Controllers**: Process requests and coordinate service calls
3. **Middleware**: Handle cross-cutting concerns (logging, error handling)
4. **Services**: Encapsulate business logic

### Data Service

**Primary Responsibility**: Manage data storage, retrieval, and processing

**Key Components**:
1. **Models**: Define data schemas and validation
2. **Repositories**: Handle database operations
3. **Services**: Implement data processing logic
4. **Utils**: Provide data transformation utilities

### Auth Service

**Primary Responsibility**: Handle user authentication and authorization

**Key Components**:
1. **Controllers**: Process auth requests
2. **Models**: Define user and permission models
3. **Services**: Implement authentication business logic
4. **Middleware**: Provide route protection

### Integration Services

**Primary Responsibility**: Connect with external systems and APIs

**Key Components**:
1. **Clients**: Interface with external APIs
2. **Adapters**: Transform between internal and external formats
3. **Services**: Implement integration business logic
4. **Utils**: Provide integration-specific utilities

## File Naming Conventions

The project follows these file naming conventions:

1. **JavaScript/TypeScript Files**:
   - `camelCase.js` for utility files
   - `PascalCase.js` for components and classes

2. **Test Files**:
   - `fileName.test.js` for unit tests
   - `fileName.spec.js` for integration tests
   - `fileName.e2e.js` for end-to-end tests

3. **Configuration Files**:
   - `configuration-name.config.js`
   - `.configname` for dot files

4. **Documentation Files**:
   - `kebab-case.md` for documentation

## Module Organization

### Controller Module Example

```javascript
/**
 * User Controller
 * Responsible for handling user-related requests
 */

// External dependencies
const express = require('express');

// Internal dependencies
const UserService = require('../services/userService');
const { asyncHandler } = require('../utils/errorHandler');
const logger = require('../utils/logger');

// Create controller logger
const controllerLogger = logger.getModuleLogger('UserController');

/**
 * Get all users with optional filtering
 */
const getUsers = asyncHandler(async (req, res) => {
  controllerLogger.debug('Getting users with filters', { filters: req.query });
  
  const users = await UserService.getUsers(req.query);
  
  return res.status(200).json({
    success: true,
    data: users
  });
});

/**
 * Get user by ID
 */
const getUserById = asyncHandler(async (req, res) => {
  const { id } = req.params;
  controllerLogger.debug('Getting user by ID', { id });
  
  const user = await UserService.getUserById(id);
  
  if (!user) {
    throw new Error('User not found');
  }
  
  return res.status(200).json({
    success: true,
    data: user
  });
});

// Export controller methods
module.exports = {
  getUsers,
  getUserById
};
```

### Service Module Example

```javascript
/**
 * User Service
 * Implements business logic for user operations
 */

// External dependencies
const mongoose = require('mongoose');

// Internal dependencies
const User = require('../models/user');
const logger = require('../utils/logger');

// Create service logger
const serviceLogger = logger.getModuleLogger('UserService');

/**
 * Get users with optional filtering
 */
const getUsers = async (filters = {}) => {
  serviceLogger.debug('Getting users with filters', { filters });
  
  // Build query
  const query = {};
  
  if (filters.name) {
    query.name = { $regex: filters.name, $options: 'i' };
  }
  
  if (filters.role) {
    query.role = filters.role;
  }
  
  // Execute query
  const users = await User.find(query)
    .limit(parseInt(filters.limit) || 10)
    .skip(parseInt(filters.page - 1) * parseInt(filters.limit) || 0)
    .sort(filters.sort || 'createdAt');
  
  serviceLogger.debug('Retrieved users', { count: users.length });
  
  return users;
};

/**
 * Get user by ID
 */
const getUserById = async (id) => {
  serviceLogger.debug('Getting user by ID', { id });
  
  if (!mongoose.Types.ObjectId.isValid(id)) {
    serviceLogger.warn('Invalid user ID format', { id });
    return null;
  }
  
  const user = await User.findById(id);
  
  if (!user) {
    serviceLogger.warn('User not found', { id });
    return null;
  }
  
  serviceLogger.debug('Retrieved user', { id, email: user.email });
  
  return user;
};

// Export service methods
module.exports = {
  getUsers,
  getUserById
};
```

## Communication Patterns

### Synchronous Communication (REST)

Services communicate synchronously using RESTful APIs for:
- Direct request-response interactions
- User-initiated actions
- Data retrieval operations

Example flow:
1. Client sends request to API Gateway
2. API Gateway routes to the appropriate service
3. Service processes request and returns response
4. API Gateway returns response to client

### Asynchronous Communication (Events)

Services communicate asynchronously using message queues for:
- Long-running processes
- Notifications and updates
- Background processing

Example flow:
1. Service A publishes an event to the message queue
2. Service B subscribes to and receives the event
3. Service B processes the event asynchronously
4. Service B publishes a result event if needed

## Database Organization

### MongoDB Collections

The project uses MongoDB with the following collection structure:

1. **users**: User accounts and profiles
   - Authentication information
   - Profile data
   - Preferences

2. **resources**: Primary business entities
   - Core attributes
   - Metadata
   - Relationship references

3. **activities**: System activity records
   - User actions
   - System events
   - Timestamps and context

4. **settings**: System configuration
   - Feature flags
   - System parameters
   - Environment-specific settings

### Indexing Strategy

The database uses the following indexing strategy:

1. **Primary Keys**: Automatic `_id` index on all collections
2. **Lookup Indexes**: Single-field indexes for frequent lookups
3. **Compound Indexes**: Multi-field indexes for common query patterns
4. **Text Indexes**: Full-text search indexes for search functionality
5. **Geospatial Indexes**: For location-based queries

## Logging and Monitoring

### Logging Standards

Each service implements standardized logging:

1. **Structured Logging**: JSON format for machine parsing
2. **Context Enrichment**: Request ID, user ID, service name
3. **Log Levels**: DEBUG, INFO, WARN, ERROR
4. **Sensitive Data Handling**: Redaction of sensitive information
5. **Performance Logging**: Timing of critical operations

### Log Format Example

```json
{
  "timestamp": "2025-05-15T12:34:56.789Z",
  "level": "info",
  "message": "User authentication successful",
  "service": "auth-service",
  "module": "AuthController",
  "requestId": "req-123-456-789",
  "userId": "user-456",
  "duration": 45,
  "method": "POST",
  "path": "/api/auth/login",
  "ip": "192.168.1.1"
}
```

## Development Workflows

### Feature Development Workflow

1. **Branch Creation**: Create a feature branch from `develop`
2. **Local Development**: Implement the feature with tests
3. **Code Review**: Submit PR and address review comments
4. **Testing**: Run tests and ensure coverage requirements are met
5. **Documentation**: Update relevant documentation
6. **Merge**: Merge into `develop` branch
7. **Deployment**: Deploy to development environment

### Bug Fix Workflow

1. **Issue Creation**: Create an issue with reproduction steps
2. **Branch Creation**: Create a bug fix branch from `develop`
3. **Fix Implementation**: Implement the fix with tests
4. **Regression Testing**: Ensure fix doesn't break existing functionality
5. **Code Review**: Submit PR and address review comments
6. **Merge**: Merge into `develop` and potentially `main` for critical fixes
7. **Deployment**: Deploy to appropriate environments

## Conclusion

This project structure is designed to ensure:

1. **Modularity**: Each component has a clear responsibility
2. **Scalability**: Services can be scaled independently
3. **Maintainability**: Code organization facilitates maintenance
4. **Testability**: Structure supports comprehensive testing
5. **Extensibility**: New features can be added with minimal changes

The structure may evolve over time to accommodate new requirements and technologies, but the core principles of separation of concerns, modularity, and clear organization will remain consistent.
