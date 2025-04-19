# Service Deployment Guide

## Overview

This guide provides comprehensive, step-by-step instructions for deploying services into the project. It covers the entire process from initial setup to production deployment, with special attention to testing requirements, common challenges, and best practices.

## Table of Contents

1. [Understanding Service Architecture](#understanding-service-architecture)
2. [Prerequisites](#prerequisites)
3. [Development Process](#development-process)
4. [Testing Requirements](#testing-requirements)
5. [Deployment Process](#deployment-process)
6. [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)
7. [Accelerating Development with LLMs](#accelerating-development-with-llms)
8. [Validation Checklist](#validation-checklist)

## Understanding Service Architecture

The system follows a microservices architecture where each service has a specific responsibility and communicates with other services through well-defined APIs. This approach provides several benefits:

1. **Modularity**: Each service has a specific responsibility, making the system easier to maintain and extend
2. **Scalability**: Services can be deployed and scaled independently
3. **Resilience**: Failures in one service do not necessarily affect others
4. **Technology Flexibility**: Different services can use different technologies as appropriate

### Service Types

The project consists of several types of services:

1. **Core API Services**: Central services that provide the main application functionality
2. **Data Processing Services**: Services focused on data transformation and analysis
3. **Integration Services**: Services that connect with external systems
4. **Supporting Services**: Infrastructure components like authentication, logging, etc.

### Communication Patterns

Services communicate using the following patterns:

1. **Synchronous REST APIs**: For direct request-response interactions
2. **Asynchronous Messaging**: For event-driven communication
3. **Shared Database Access**: Limited to specific service groups with clear ownership

## Prerequisites

### Environment Setup

1. **Required Software**
   - Node.js (LTS version)
   - Docker and Docker Compose
   - Git
   - Database tools appropriate for your service

2. **Platform-Specific Configuration**
   - **Windows**: Docker Desktop with WSL2 integration
   - **macOS**: Docker Desktop with 4GB+ memory allocation
   - **Linux**: Docker Engine and Docker Compose

3. **Verify Environment**
   - Ensure Docker daemon is running: `docker info`
   - Verify Node.js version: `node -v`
   - Check system date/time (affects logging timestamps)

## Development Process

### Step 1: Initial Service Structure

1. **Create Service Directory**
   ```
   mkdir service-[name]
   cd service-[name]
   npm init -y
   ```

2. **Set Up Directory Structure**
   ```
   mkdir -p controllers middleware models routes tests/{unit,integration,e2e} utils scripts
   ```

3. **Install Core Dependencies**
   ```bash
   npm install express mongoose winston dotenv
   npm install --save-dev jest supertest mongodb-memory-server nodemon cross-env
   ```

4. **Create Base Files**

   Create the following essential files:
   - `.env` - Environment variables
   - `.gitignore` - Git ignore rules
   - `app.js` - Express application setup
   - `server.js` - Server initialization
   - `jest.config.js` - Jest configuration
   - `Dockerfile` - Container definition
   - `Dockerfile.test` - Test container definition

### Step 2: Implement Core Service Components

1. **Logger Setup** (utils/logger.js)
   ```javascript
   const winston = require('winston');
   
   // Create logger with standard format
   const logger = winston.createLogger({
     level: process.env.LOG_LEVEL || 'info',
     format: winston.format.combine(
       winston.format.timestamp(),
       winston.format.errors({ stack: true }),
       winston.format.json()
     ),
     defaultMeta: { service: 'service-[name]' },
     transports: [
       new winston.transports.Console(),
       new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
       new winston.transports.File({ filename: 'logs/combined.log' })
     ]
   });
   
   // Create module-specific logger
   logger.getModuleLogger = (module) => {
     return logger.child({ module });
   };
   
   // Create function wrapper for logging
   logger.wrapWithLogging = (fn, options = {}) => {
     const moduleLogger = options.module ? 
       logger.getModuleLogger(options.module) : logger;
     const functionName = options.functionName || fn.name || 'anonymous';
     
     return async (...args) => {
       try {
         moduleLogger.debug(`${functionName} started`, { 
           functionName,
           params: args
         });
         const startTime = Date.now();
         const result = await fn(...args);
         const duration = Date.now() - startTime;
         
         moduleLogger.debug(`${functionName} completed`, { 
           functionName,
           duration,
           result
         });
         return result;
       } catch (error) {
         moduleLogger.error(`${functionName} failed`, { 
           functionName,
           error
         });
         throw error;
       }
     };
   };
   
   module.exports = logger;
   ```

2. **Error Handler** (utils/errorHandler.js)
   ```javascript
   // Custom API error class
   class ApiError extends Error {
     constructor(statusCode, message, details = null) {
       super(message);
       this.statusCode = statusCode;
       this.details = details;
     }
   }
   
   // Async handler wrapper
   const asyncHandler = (fn) => (req, res, next) => {
     Promise.resolve(fn(req, res, next)).catch(next);
   };
   
   // Not found error generator
   const notFound = (resource) => {
     return new ApiError(404, `${resource} not found`);
   };
   
   // Error handling middleware
   const handleError = (err, req, res, next) => {
     const statusCode = err.statusCode || 500;
     const message = err.message || 'Internal Server Error';
     
     // Log the error
     const logger = require('./logger');
     if (statusCode >= 500) {
       logger.error('Server error', {
         error: err.message,
         stack: err.stack,
         url: req.originalUrl,
         method: req.method,
         requestId: req.requestId
       });
     } else {
       logger.warn('Client error', {
         statusCode,
         message,
         url: req.originalUrl,
         method: req.method,
         requestId: req.requestId
       });
     }
     
     res.status(statusCode).json({
       success: false,
       error: {
         message,
         details: err.details || undefined,
         ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
       }
     });
   };
   
   module.exports = {
     ApiError,
     asyncHandler,
     notFound,
     handleError
   };
   ```

3. **Express App Setup** (app.js)
   ```javascript
   const express = require('express');
   const logger = require('./utils/logger');
   const { handleError } = require('./utils/errorHandler');
   
   // Create Express app
   const app = express();
   
   // Middleware
   app.use(express.json());
   app.use(require('./middleware/requestLogger'));
   
   // Health check endpoint
   app.get('/health', (req, res) => {
     res.status(200).json({ status: 'ok' });
   });
   
   // API routes
   app.use('/api/[resource]', require('./routes/[resourceRoutes]'));
   
   // Error handling 
   app.use((req, res, next) => {
     next(new Error('Endpoint not found'));
   });
   app.use(handleError);
   
   module.exports = app;
   ```

4. **Server Initialization** (server.js)
   ```javascript
   const app = require('./app');
   const mongoose = require('mongoose');
   const logger = require('./utils/logger');
   
   // Environment variables
   const PORT = process.env.PORT || 4000;
   const NODE_ENV = process.env.NODE_ENV || 'development';
   
   // Determine MongoDB URI
   let MONGO_URI;
   if (NODE_ENV === 'test' || NODE_ENV === 'e2e-test') {
     MONGO_URI = process.env.MONGO_URI_TEST || 'mongodb://mongodb-test:27017/[service-name]-test';
   } else {
     MONGO_URI = process.env.MONGO_URI || 'mongodb://localhost:27017/[service-name]';
   }
   
   // Create server
   const server = require('http').createServer(app);
   
   // Track connection state
   let isConnectedToMongo = false;
   
   // Connect to MongoDB with retry
   async function connectWithRetry(uri, options = {}, maxRetries = 5, delay = 5000) {
     let retries = 0;
     while (retries < maxRetries) {
       try {
         await mongoose.connect(uri, options);
         isConnectedToMongo = true;
         logger.info('Connected to MongoDB successfully');
         return true;
       } catch (err) {
         retries++;
         logger.warn(`MongoDB connection attempt ${retries}/${maxRetries} failed`, { 
           error: err.message 
         });
         await new Promise(resolve => setTimeout(resolve, delay));
       }
     }
     logger.error(`Failed to connect to MongoDB after ${maxRetries} attempts`);
     return false;
   }
   
   // Start server
   async function startServer() {
     try {
       const connected = await connectWithRetry(MONGO_URI, {
         useNewUrlParser: true,
         useUnifiedTopology: true,
         serverSelectionTimeoutMS: 10000,
         socketTimeoutMS: 45000,
       });
   
       if (!connected) {
         logger.error('Cannot start server without database connection');
         process.exit(1);
       }
   
       server.listen(PORT, () => {
         logger.info(`Server running in ${NODE_ENV} mode on port ${PORT}`);
       });
     } catch (error) {
       logger.error('Server startup error', { error: error.message, stack: error.stack });
       process.exit(1);
     }
   }
   
   // Graceful shutdown
   function gracefulShutdown(signal) {
     return async () => {
       logger.info(`${signal} received, shutting down gracefully...`);
       
       server.close(() => {
         logger.info('HTTP server closed');
         
         if (isConnectedToMongo) {
           mongoose.connection.close(false)
             .then(() => {
               logger.info('MongoDB connection closed');
               process.exit(0);
             })
             .catch((err) => {
               logger.error('Error closing MongoDB connection', { 
                 error: err.message 
               });
               process.exit(1);
             });
         } else {
           process.exit(0);
         }
       });
       
       // Force exit after timeout
       setTimeout(() => {
         logger.error('Forcing shutdown after timeout');
         process.exit(1);
       }, 10000);
     };
   }
   
   // Signal handlers
   process.on('SIGTERM', gracefulShutdown('SIGTERM'));
   process.on('SIGINT', gracefulShutdown('SIGINT'));
   
   // Unhandled rejections and exceptions
   process.on('unhandledRejection', (reason, promise) => {
     logger.error('Unhandled Promise Rejection', {
       error: reason instanceof Error ? reason.message : reason,
       stack: reason instanceof Error ? reason.stack : undefined,
     });
   });
   
   process.on('uncaughtException', (error) => {
     logger.error('Uncaught Exception', {
       error: error.message,
       stack: error.stack,
     });
   });
   
   // Start server
   startServer();
   
   // Export for testing
   module.exports = server;
   ```

### Step 3: Create Docker Configuration

1. **Dockerfile**
   ```dockerfile
   FROM node:lts-alpine
   
   WORKDIR /app
   
   COPY package*.json ./
   
   RUN npm ci
   
   COPY . .
   
   # Create log directory
   RUN mkdir -p logs
   
   # Health check
   HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
     CMD wget -qO- http://localhost:${PORT:-4000}/health || exit 1
   
   EXPOSE ${PORT:-4000}
   
   CMD ["node", "server.js"]
   ```

2. **Dockerfile.test**
   ```dockerfile
   FROM node:lts-alpine
   
   WORKDIR /app
   
   COPY package*.json ./
   
   RUN npm ci
   
   COPY . .
   
   # Create log directory
   RUN mkdir -p logs
   
   # Set environment variables
   ENV NODE_ENV=test
   
   # No need to expose port for tests
   
   CMD ["npm", "test"]
   ```

3. **Add to docker-compose.yml** (root project)
   ```yaml
   services:
     # ... existing services
     
     service-[name]:
       build:
         context: ./service-[name]
       ports:
         - "[port]:4000"
       environment:
         - NODE_ENV=development
         - PORT=4000
         - MONGO_URI=mongodb://mongodb:27017/[database]
       volumes:
         - ./service-[name]/logs:/app/logs
       depends_on:
         mongodb:
           condition: service_healthy
       healthcheck:
         test: ["CMD", "wget", "-qO-", "http://localhost:4000/health"]
         interval: 30s
         timeout: 10s
         retries: 3
         start_period: 30s
   ```

4. **Add to docker-compose.test.yml** (root project)
   ```yaml
   services:
     # ... existing services
     
     service-[name]-test:
       build:
         context: ./service-[name]
         dockerfile: Dockerfile.test
       environment:
         - NODE_ENV=test
         - MONGO_URI=mongodb://mongodb-test:27017/[service-name]-test
       volumes:
         - ./service-[name]/logs:/app/logs
       depends_on:
         mongodb-test:
           condition: service_healthy
   ```

### Step 4: Implement Cross-Platform Test Runners

1. **Create Unix Shell Script** (scripts/run-e2e-tests.sh)
   ```bash
   #!/bin/bash
   set -e
   
   SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
   PROJECT_DIR="$(dirname "$SCRIPT_DIR")"
   
   cd "$PROJECT_DIR"
   NODE_ENV=e2e-test npx jest --config=jest.e2e.config.js "$@"
   ```

2. **Create Windows PowerShell Script** (scripts/run-e2e-tests-windows.ps1)
   ```powershell
   $ErrorActionPreference = "Stop"
   
   $scriptDir = Split-Path -Parent $MyInvocation.MyCommand.Path
   $projectDir = Split-Path -Parent $scriptDir
   
   Set-Location $projectDir
   $env:NODE_ENV = "e2e-test"
   npx jest --config=jest.e2e.config.js $args
   exit $LASTEXITCODE
   ```

3. **Make Scripts Executable** (Unix-based systems)
   ```bash
   chmod +x scripts/run-e2e-tests.sh
   ```

4. **Update package.json Scripts**
   ```json
   "scripts": {
     "start": "node server.js",
     "dev": "nodemon server.js",
     "test": "jest --config=jest.config.js",
     "test:watch": "jest --watch",
     "test:coverage": "jest --coverage",
     "test:e2e": "cross-env NODE_ENV=e2e-test jest --config=jest.e2e.config.js",
     "test:e2e:unix": "./scripts/run-e2e-tests.sh",
     "test:e2e:windows": "powershell ./scripts/run-e2e-tests-windows.ps1",
     "lint": "eslint ."
   }
   ```

## Testing Requirements

### Step 5: Unit Testing Requirements (100% Coverage)

For each service, implement comprehensive unit tests covering:

1. **Models**
   - Schema validation (required fields, defaults, enums)
   - Custom methods and virtual properties
   - Pre/post hooks and middleware
   - Indexing and unique constraints

2. **Controllers**
   - Response status codes
   - Response body format
   - Error handling
   - Parameter validation
   - Pagination and filtering

3. **Middleware**
   - Request/response modifications
   - Error handling
   - Authentication/authorization logic

4. **Utilities**
   - Logger functionality
   - Error handling utilities
   - Helper functions
   - Data transformations

Example unit test structure:

```javascript
// Sample model test
describe('MyModel', () => {
  beforeAll(async () => {
    // Setup test database connection
  });
  
  afterAll(async () => {
    // Clean up connection
  });
  
  beforeEach(async () => {
    // Clear collection before each test
    await MyModel.deleteMany({});
  });
  
  describe('Validation', () => {
    it('should validate required fields', async () => {
      const missingRequired = new MyModel({ /* missing required fields */ });
      await expect(missingRequired.save()).rejects.toThrow();
    });
    
    it('should accept valid data', async () => {
      const validModel = new MyModel({ /* valid test data */ });
      const savedModel = await validModel.save();
      expect(savedModel._id).toBeDefined();
    });
  });
  
  // More test groups...
});
```

### Step 6: Integration Testing Requirements

All services must implement integration tests for:

1. **Database Operations**
   - CRUD operations work correctly with real database
   - Queries return expected results
   - Indexes are properly used

2. **API Integration**
   - Routes are properly connected to controllers
   - Middleware chain works correctly
   - Response body conforms to API specifications

3. **Service Integration**
   - Service can connect with other dependent services
   - Error handling for service communication
   - Proper data transformation between services

Example integration test structure:

```javascript
// Sample API endpoint integration test
describe('Resource API', () => {
  let token;
  
  beforeAll(async () => {
    // Setup for all tests
    await mongoose.connect(process.env.MONGO_URI_TEST);
    // Get authentication token if needed
    token = await getTestToken();
  });
  
  afterAll(async () => {
    // Cleanup after all tests
    await mongoose.connection.close();
  });
  
  beforeEach(async () => {
    // Setup before each test
    await Resource.deleteMany({});
    // Create test data
    await Resource.create([
      { name: 'Test 1', type: 'category1' },
      { name: 'Test 2', type: 'category2' }
    ]);
  });
  
  describe('GET /api/resources', () => {
    it('should get all resources', async () => {
      const res = await request(app)
        .get('/api/resources')
        .set('Authorization', `Bearer ${token}`);
      
      expect(res.statusCode).toEqual(200);
      expect(res.body.success).toBe(true);
      expect(Array.isArray(res.body.data)).toBe(true);
      expect(res.body.data.length).toBe(2);
    });
    
    it('should paginate results correctly', async () => {
      const res = await request(app)
        .get('/api/resources?limit=1&page=1')
        .set('Authorization', `Bearer ${token}`);
      
      expect(res.statusCode).toEqual(200);
      expect(res.body.success).toBe(true);
      expect(res.body.data.length).toBe(1);
      expect(res.body.pagination).toBeDefined();
      expect(res.body.pagination.totalPages).toBe(2);
      expect(res.body.pagination.currentPage).toBe(1);
    });
    
    // More endpoint tests...
  });
});
```

### Step 7: End-to-End Testing Requirements

Implement E2E tests that validate:

1. **Complete Workflows**
   - Test entire business process flows
   - Test with realistic data scenarios
   - Test cross-service interactions

2. **Performance Testing**
   - Response times meet requirements
   - System handles expected load
   - Resources are properly released

3. **Error Recovery**
   - System recovers from service failures
   - Proper fallback mechanisms
   - Error messages are clear and helpful

Example E2E test structure:

```javascript
// Sample E2E test
describe('Resource Management Workflow', () => {
  beforeAll(async () => {
    // Start required services
    await startTestServices();
  });
  
  afterAll(async () => {
    // Shut down services
    await cleanupTestServices();
  });
  
  it('should create and update a resource through the complete workflow', async () => {
    // Step 1: Create a resource
    const createResponse = await request(baseUrl)
      .post('/api/resources')
      .set('Authorization', `Bearer ${token}`)
      .send({
        name: 'Test Resource',
        type: 'test',
        attributes: {
          key1: 'value1',
          key2: 'value2'
        }
      });
    
    expect(createResponse.status).toBe(201);
    expect(createResponse.body.success).toBe(true);
    const resourceId = createResponse.body.data.id;
    
    // Step 2: Retrieve the resource
    const getResponse = await request(baseUrl)
      .get(`/api/resources/${resourceId}`)
      .set('Authorization', `Bearer ${token}`);
    
    expect(getResponse.status).toBe(200);
    expect(getResponse.body.data.name).toBe('Test Resource');
    
    // Step 3: Update the resource
    const updateResponse = await request(baseUrl)
      .put(`/api/resources/${resourceId}`)
      .set('Authorization', `Bearer ${token}`)
      .send({
        name: 'Updated Resource',
        type: 'test',
        attributes: {
          key1: 'new-value1',
          key2: 'new-value2'
        }
      });
    
    expect(updateResponse.status).toBe(200);
    
    // Step 4: Verify the update
    const verifyResponse = await request(baseUrl)
      .get(`/api/resources/${resourceId}`)
      .set('Authorization', `Bearer ${token}`);
    
    expect(verifyResponse.status).toBe(200);
    expect(verifyResponse.body.data.name).toBe('Updated Resource');
    expect(verifyResponse.body.data.attributes.key1).toBe('new-value1');
    
    // Step 5: Delete the resource
    const deleteResponse = await request(baseUrl)
      .delete(`/api/resources/${resourceId}`)
      .set('Authorization', `Bearer ${token}`);
    
    expect(deleteResponse.status).toBe(200);
    
    // Step 6: Verify deletion
    const notFoundResponse = await request(baseUrl)
      .get(`/api/resources/${resourceId}`)
      .set('Authorization', `Bearer ${token}`);
    
    expect(notFoundResponse.status).toBe(404);
  });
  
  // More E2E tests...
});
```

## Deployment Process

### Step 8: Documentation Requirements

1. **Install Swagger Dependencies**
   ```bash
   npm install swagger-jsdoc swagger-ui-express
   ```

2. **Configure Swagger in app.js**
   ```javascript
   const swaggerJsdoc = require('swagger-jsdoc');
   const swaggerUi = require('swagger-ui-express');
   
   // Swagger definition
   const swaggerOptions = {
     definition: {
       openapi: '3.0.0',
       info: {
         title: 'Service API Documentation',
         version: '1.0.0',
         description: 'API documentation for the service',
       },
       servers: [
         {
           url: process.env.API_URL || 'http://localhost:4000',
           description: 'Development server',
         }
       ],
     },
     // Path to the API docs
     apis: ['./routes/*.js', './models/*.js'],
   };
   
   const swaggerDocs = swaggerJsdoc(swaggerOptions);
   
   // Swagger middleware
   app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocs));
   ```

3. **Document Routes with JSDoc**
   ```javascript
   /**
    * @swagger
    * /api/resources:
    *   get:
    *     summary: Retrieve a list of resources
    *     description: Returns a paginated list of resources with optional filtering
    *     tags: [Resources]
    *     parameters:
    *       - in: query
    *         name: limit
    *         schema:
    *           type: integer
    *         description: Maximum number of resources to return (default: 10)
    *       - in: query
    *         name: page
    *         schema:
    *           type: integer
    *         description: Page number (default: 1)
    *       - in: query
    *         name: type
    *         schema:
    *           type: string
    *         description: Filter by resource type
    *     responses:
    *       200:
    *         description: A list of resources
    *         content:
    *           application/json:
    *             schema:
    *               type: object
    *               properties:
    *                 success:
    *                   type: boolean
    *                   example: true
    *                 data:
    *                   type: array
    *                   items:
    *                     $ref: '#/components/schemas/Resource'
    *                 pagination:
    *                   type: object
    *                   properties:
    *                     totalItems:
    *                       type: integer
    *                     totalPages:
    *                       type: integer
    *                     currentPage:
    *                       type: integer
    *                     limit:
    *                       type: integer
    */
   router.get('/', asyncHandler(async (req, res) => {
     // Route implementation
   }));
   ```

### Step 9: Production Deployment Preparation

1. **Environment Configuration**
   - Create separate environment configurations (development, test, production)
   - Implement secure handling of environment variables
   - Document all required environment variables

2. **Security Hardening**
   - Implement security headers (Helmet.js)
   - Set up rate limiting for public endpoints
   - Configure CORS appropriately
   - Implement validation for all inputs

3. **Performance Optimization**
   - Implement database indexes for frequent queries
   - Add caching for appropriate resources
   - Configure compression middleware
   - Set up connection pooling

4. **Monitoring and Logging**
   - Configure centralized logging
   - Set up application monitoring
   - Implement health check endpoints
   - Create alerts for critical errors

## Common Issues and Troubleshooting

### Database Connection Issues

**Problem**: Connection to database fails during startup or tests.

**Solutions**:
1. **Check Connection String**
   - Verify the connection string format is correct
   - Check if the database server is running and accessible

2. **Implement Connection Retry Logic**
   ```javascript
   async function connectWithRetry(maxRetries = 5, delay = 5000) {
     let retries = 0;
     while (retries < maxRetries) {
       try {
         await mongoose.connect(MONGO_URI, mongoOptions);
         logger.info('Connected to database');
         return true;
       } catch (err) {
         retries++;
         logger.warn(`Database connection attempt ${retries}/${maxRetries} failed`);
         await new Promise(resolve => setTimeout(resolve, delay));
       }
     }
     logger.error(`Failed to connect to database after ${maxRetries} attempts`);
     return false;
   }
   ```

3. **Check Firewall/Network**
   - Verify network access between service and database
   - Check firewall rules and security groups

### Docker Container Exit Issues

**Problem**: Docker containers exit with non-zero status code.

**Solutions**:
1. **Check Container Logs**
   ```bash
   docker logs [container-id]
   ```

2. **Improve Error Handling**
   - Add proper error logging in server.js
   - Implement graceful shutdown for error scenarios

3. **Implement Health Checks**
   - Add Docker HEALTHCHECK in Dockerfile
   - Adjust healthcheck intervals in docker-compose.yml

### Cross-Platform Compatibility

**Problem**: Tests fail on Windows but work on Unix-based systems.

**Solutions**:
1. **Use Path Module for File Paths**
   ```javascript
   const path = require('path');
   const filePath = path.join(__dirname, 'data', 'file.txt');
   ```

2. **Create Platform-Specific Scripts**
   - Use .sh files for Unix-based systems
   - Use .ps1 files for Windows
   - Use cross-env for environment variables

3. **Avoid Bash-Specific Commands**
   - Replace commands like `&&` with semicolons in package.json scripts
   - Use cross-platform Node scripts instead of shell scripts when possible

### Resource Leaks in Tests

**Problem**: Jest tests hang or timeout due to unclosed resources.

**Solutions**:
1. **Implement Proper Cleanup**
   ```javascript
   // In global teardown
   async function teardown() {
     // Close database connections
     await mongoose.connection.close();
     
     // Close logger transports
     const logger = require('./utils/logger');
     if (typeof logger.closeAll === 'function') {
       await logger.closeAll();
     }
     
     // Clean up other resources
     // ...
   }
   ```

2. **Add Logger Cleanup Function**
   ```javascript
   // In logger.js
   logger.closeAll = async function() {
     return new Promise((resolve, reject) => {
       let pendingTransports = 0;
       let transportsClosed = 0;
       
       // Count transports that need to be closed
       this.transports.forEach((transport) => {
         if (transport.close) {
           pendingTransports++;
         }
       });
       
       if (pendingTransports === 0) {
         return resolve();
       }
       
       // Close each transport
       this.transports.forEach((transport) => {
         if (transport.close) {
           transport.close(() => {
             transportsClosed++;
             if (transportsClosed === pendingTransports) {
               resolve();
             }
           });
         }
       });
       
       // Timeout after 5 seconds
       setTimeout(() => {
         if (transportsClosed < pendingTransports) {
           console.warn(`Warning: Only ${transportsClosed}/${pendingTransports} transports closed properly`);
           resolve();
         }
       }, 5000);
     });
   };
   ```

## Accelerating Development with LLMs

Building services can be significantly accelerated by leveraging Large Language Models (LLMs) like Claude. This approach enables rapid prototyping, implementation, and iteration of services by using AI to generate code, design APIs, and develop integration patterns.

### Step 1: Preparing Documentation for LLMs

To effectively use LLMs for development, start by providing comprehensive contextual information:

1. **Gather Key Documentation**
   - Project architecture and design documents
   - Existing service documentation
   - API specifications
   - Database schemas

2. **Format Documentation for LLM Input**
   - Condense the most relevant sections
   - Include specific code examples from existing implementations
   - Ensure code examples show correct patterns for:
     - API endpoint implementation
     - Error handling
     - Database interaction
     - Testing approach

3. **Provide Project-Specific Context**
   ```
   The system uses the following patterns:
   - RESTful API design with consistent response format
   - Centralized error handling with error classification
   - Winston-based logging with structured format
   - Mongoose for MongoDB interactions
   - Jest for testing with separate configurations for unit/integration tests
   ```

### Step 2: Describing Your Service Requirements

When using an LLM to build your service, clearly articulate requirements:

1. **Service Specification Template**
   ```
   I need a service that:
   - Connects to [database/external system]
   - Exposes [specific resources] through RESTful APIs
   - Implements [specific business logic]
   - Follows the project architecture and patterns
   - Includes proper error handling and logging
   - Has comprehensive test coverage
   ```

2. **Example Service Description**
   ```
   Build a service that:
   - Connects to MongoDB to store and retrieve user data
   - Provides CRUD operations for user profiles
   - Implements user search and filtering
   - Follows the project's RESTful API patterns
   - Includes comprehensive logging and error handling
   - Implements all required testing (unit, integration, E2E)
   ```

3. **Include Specific Implementation Details**
   - Database schema or API specifications
   - Authentication requirements
   - Performance expectations
   - Specific error handling needs
   - Integration points with other services

### Step 3: Working with LLMs to Build the Service

Follow these steps to effectively collaborate with LLMs for service development:

1. **Start with Core Components**
   - Begin with the basic server setup
   - Focus on key model definitions
   - Get the core API endpoints working first
   - Validate against the project standards

2. **Iterative Implementation**
   - Ask the LLM to implement one component at a time
   - Review each component before moving to the next
   - Request explanations for complex parts
   - Iterate on components that don't meet requirements

3. **Code Review and Refinement**
   - Ask the LLM to review its own generated code
   - Request optimization suggestions
   - Identify potential edge cases
   - Improve error handling and logging

4. **Example Prompt for Implementation**
   ```
   Based on the project documentation provided, implement the User model and basic CRUD controllers in Express.js. Include:
   1. Mongoose schema with validation
   2. Controller methods for creating, reading, updating, and deleting users
   3. Proper error handling following project standards
   4. Logging integration with context tracking
   5. Input validation middleware
   ```

### Step 4: Testing and Documentation with LLM Assistance

LLMs can also help with testing and documenting your service:

1. **Generating Test Cases**
   - Ask the LLM to generate comprehensive test cases
   - Focus on edge cases and error scenarios
   - Request tests for each controller and model
   - Ensure proper setup and teardown in tests

2. **Documentation Generation**
   - Request API documentation in Swagger/OpenAPI format
   - Generate detailed README.md content
   - Create usage examples
   - Document error codes and troubleshooting

3. **Example Prompt for Testing**
   ```
   Generate unit tests for the User controller implemented in our service. Include tests for:
   1. Successful CRUD operations
   2. Validation error handling
   3. Not found scenarios
   4. Authentication/authorization failures
   5. Edge cases (invalid IDs, malformed requests, etc.)
   ```

## Validation Checklist

Before considering a service ready for production, ensure it meets these criteria:

### 1. Code Quality
- [ ] Passes all linting rules
- [ ] No console.log statements (use logger instead)
- [ ] All functions have JSDoc documentation
- [ ] Complex functions include explanatory comments
- [ ] No hard-coded configuration (use environment variables)

### 2. Test Coverage
- [ ] 100% unit test coverage for critical components
- [ ] Integration tests for all API endpoints
- [ ] E2E tests for key workflows
- [ ] Performance tests meet response time requirements
- [ ] Edge cases and error scenarios are tested

### 3. Logging
- [ ] Comprehensive logging with proper log levels
- [ ] Function entry/exit logging for complex operations
- [ ] Error logging with stack traces
- [ ] Performance metrics for significant operations
- [ ] Request/response logging with proper sanitization

### 4. Documentation
- [ ] Complete API documentation with Swagger/OpenAPI
- [ ] README.md with setup and usage instructions
- [ ] Environment variable documentation
- [ ] DEVLOG.md updated with implementation details
- [ ] Code comments explaining complex logic

### 5. Docker Configuration
- [ ] Dockerfile optimized for production
- [ ] Docker Compose configuration
- [ ] Health checks properly implemented
- [ ] Resource limits configured
- [ ] Volume mounts for persistent data

### 6. Security
- [ ] Input validation on all API endpoints
- [ ] No sensitive data in logs or responses
- [ ] Proper error handling without leaking details
- [ ] Authentication/authorization where needed
- [ ] Secure dependency versions (no CVEs)

### 7. Deployment Readiness
- [ ] Service registered with service registry
- [ ] Health endpoint correctly reports status
- [ ] Graceful shutdown handling
- [ ] Resource cleanup on termination
- [ ] No test-only code in production build

---

By following this guide, you will create robust, well-tested services that integrate seamlessly with the project ecosystem. Remember to document any challenges encountered during implementation in the DEVLOG.md file to help future developers.
