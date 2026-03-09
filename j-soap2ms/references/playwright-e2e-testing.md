# Playwright E2E Testing Framework

## Overview

Playwright provides a robust framework for end-to-end testing of microservices APIs and user workflows. This guide creates a comprehensive E2E test suite for validating order management workflows across microservices.

## Architecture

```
Playwright Test Suite
├── Tests (test files with scenarios)
├── Page Objects (API client abstractions)
├── Fixtures (shared setup/teardown)
├── Data Factories (generate test data)
└── Reporters (HTML, JUnit, custom)
```

## Step 1: Initialize Playwright Project

```bash
# Create directory for E2E tests
mkdir e2e-tests
cd e2e-tests

# Initialize npm/Node.js
npm init -y

# Install Playwright
npm install -D @playwright/test typescript ts-node

# Install utilities
npm install -D dotenv axios uuid @faker-js/faker

# Create tsconfig.json
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["tests/**/*", "pages/**/*", "fixtures/**/*"]
}
EOF

# Initialize Playwright configuration
npx playwright install
```

## Step 2: Create Playwright Configuration

### playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  
  // Test configuration
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  
  // Output
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'test-results/results.xml' }],
    ['list'],
  ],
  
  // Shared settings for all webServers
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:8080',
    httpCredentials: process.env.AUTH_TOKEN ? {
      username: 'api_user',
      password: process.env.AUTH_TOKEN,
    } : undefined,
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  
  // Run your local dev server before starting the tests
  webServer: {
    command: 'mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=test"',
    url: 'http://localhost:8080/actuator/health',
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
  
  // Configure projects for major browsers
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
  
  // Global timeout
  timeout: 30000,
  expect: {
    timeout: 5000,
  },
});
```

## Step 3: Create Page Objects (API Client Abstractions)

### pages/api-gateway.page.ts

```typescript
import { APIRequestContext, expect } from '@playwright/test';

export interface OrderRequest {
  customerId: string;
  items: OrderItem[];
  shippingAddress: Address;
}

export interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

export interface Address {
  street: string;
  city: string;
  state: string;
  zip: string;
}

export interface OrderResponse {
  orderId: string;
  customerId: string;
  status: string;
  items: OrderItem[];
  totalAmount: number;
  createdDate: string;
}

export class ApiGatewayPage {
  private baseURL: string;
  private request: APIRequestContext;

  constructor(request: APIRequestContext, baseURL: string = 'http://localhost:8080') {
    this.request = request;
    this.baseURL = baseURL;
  }

  // Create Order
  async createOrder(orderData: OrderRequest): Promise<OrderResponse> {
    const response = await this.request.post(`${this.baseURL}/api/v1/orders`, {
      data: orderData,
    });

    expect(response.status()).toBe(201);
    return response.json() as Promise<OrderResponse>;
  }

  // Get Order
  async getOrder(orderId: string): Promise<OrderResponse> {
    const response = await this.request.get(`${this.baseURL}/api/v1/orders/${orderId}`);

    expect(response.status()).toBe(200);
    return response.json() as Promise<OrderResponse>;
  }

  // Update Order
  async updateOrder(orderId: string, status: string): Promise<OrderResponse> {
    const response = await this.request.patch(`${this.baseURL}/api/v1/orders/${orderId}`, {
      data: { status },
    });

    expect(response.status()).toBe(200);
    return response.json() as Promise<OrderResponse>;
  }

  // Cancel Order
  async cancelOrder(orderId: string): Promise<void> {
    const response = await this.request.delete(`${this.baseURL}/api/v1/orders/${orderId}`);

    expect(response.status()).toBe(204);
  }

  // List Orders
  async listOrders(customerId?: string): Promise<OrderResponse[]> {
    const params = customerId ? { customerId } : {};
    const response = await this.request.get(`${this.baseURL}/api/v1/orders`, {
      params,
    });

    expect(response.status()).toBe(200);
    return response.json() as Promise<OrderResponse[]>;
  }

  // Check product availability
  async checkAvailability(productId: string, quantity: number): Promise<boolean> {
    const response = await this.request.get(
      `${this.baseURL}/api/v1/products/${productId}/availability`,
      {
        params: { quantity },
      }
    );

    expect(response.status()).toBe(200);
    const data = await response.json();
    return data.available;
  }
}
```

### pages/inventory.page.ts

```typescript
import { APIRequestContext, expect } from '@playwright/test';

export interface ProductResponse {
  productId: string;
  name: string;
  currentStock: number;
  reservedStock: number;
  availableStock: number;
}

export class InventoryPage {
  private baseURL: string;
  private request: APIRequestContext;

  constructor(request: APIRequestContext, baseURL: string = 'http://localhost:8080') {
    this.request = request;
    this.baseURL = baseURL;
  }

  async getProduct(productId: string): Promise<ProductResponse> {
    const response = await this.request.get(
      `${this.baseURL}/api/v1/products/${productId}`
    );

    expect(response.status()).toBe(200);
    return response.json() as Promise<ProductResponse>;
  }

  async reserveStock(productId: string, quantity: number, orderId: string): Promise<void> {
    const response = await this.request.post(
      `${this.baseURL}/api/v1/products/${productId}/reserve`,
      {
        data: { quantity, orderId },
      }
    );

    expect(response.status()).toBe(200);
  }

  async releaseStock(productId: string, quantity: number, orderId: string): Promise<void> {
    const response = await this.request.post(
      `${this.baseURL}/api/v1/products/${productId}/release`,
      {
        data: { quantity, orderId },
      }
    );

    expect(response.status()).toBe(200);
  }

  async getAvailableStock(productId: string): Promise<number> {
    const product = await this.getProduct(productId);
    return product.availableStock;
  }
}
```

## Step 4: Create Test Fixtures (Setup/Teardown)

### fixtures/auth.fixture.ts

```typescript
import { test as base, APIRequestContext } from '@playwright/test';
import axios from 'axios';

export type AuthFixtures = {
  authenticatedRequest: APIRequestContext;
  token: string;
};

export const test = base.extend<AuthFixtures>({
  token: [async ({}, use) => {
    // Get JWT token from auth service
    const authResponse = await axios.post(
      process.env.AUTH_SERVICE_URL || 'http://localhost:9000/auth/login',
      {
        username: process.env.TEST_USERNAME || 'test_user',
        password: process.env.TEST_PASSWORD || 'test_password',
      }
    );

    const token = authResponse.data.access_token;
    await use(token);
  }, { scope: 'test' }],

  authenticatedRequest: async ({ request, token }, use) => {
    // Create request context with auth header
    const authenticatedRequest = request;
    
    // Set default headers
    await use(authenticatedRequest);
  },
});

export { expect } from '@playwright/test';
```

### fixtures/database.fixture.ts

```typescript
import { test as base } from '@playwright/test';
import * as oracledb from 'oracledb';

export type DatabaseFixtures = {
  db: DatabaseHelper;
};

export class DatabaseHelper {
  private connection: any;

  async connect() {
    this.connection = await oracledb.getConnection({
      user: process.env.DB_USERNAME || 'test_user',
      password: process.env.DB_PASSWORD || 'test_pwd',
      connectionString:
        process.env.DB_CONNECT_STRING || 'localhost:1521/XEPDB1',
    });
  }

  async insertTestData(table: string, data: any) {
    const columns = Object.keys(data).join(', ');
    const values = Object.values(data)
      .map((v) => `'${v}'`)
      .join(', ');
    const sql = `INSERT INTO ${table} (${columns}) VALUES (${values})`;
    await this.connection.execute(sql);
    await this.connection.commit();
  }

  async clearTable(table: string) {
    await this.connection.execute(`DELETE FROM ${table}`);
    await this.connection.commit();
  }

  async cleanup() {
    await this.connection.close();
  }
}

export const test = base.extend<DatabaseFixtures>({
  db: async ({}, use) => {
    const db = new DatabaseHelper();
    await db.connect();
    await use(db);
    await db.cleanup();
  },
});

export { expect } from '@playwright/test';
```

## Step 5: Create Test Scenarios

### tests/order-flows.spec.ts

```typescript
import { test, expect } from '../fixtures/auth.fixture';
import { ApiGatewayPage, OrderRequest } from '../pages/api-gateway.page';
import { v4 as uuidv4 } from 'uuid';

test.describe('Order Management Flows', () => {
  let apiGateway: ApiGatewayPage;
  let testCustomerId: string;

  test.beforeEach(async ({ request }) => {
    apiGateway = new ApiGatewayPage(request);
    testCustomerId = uuidv4();
  });

  test('should create an order successfully', async () => {
    // Arrange: Create test order data
    const orderData: OrderRequest = {
      customerId: testCustomerId,
      items: [
        {
          productId: 'PROD-001',
          quantity: 2,
          price: 29.99,
        },
      ],
      shippingAddress: {
        street: '123 Main St',
        city: 'Springfield',
        state: 'IL',
        zip: '62701',
      },
    };

    // Act: Create order
    const order = await apiGateway.createOrder(orderData);

    // Assert: Verify order created successfully
    expect(order).toBeDefined();
    expect(order.orderId).toBeTruthy();
    expect(order.customerId).toBe(testCustomerId);
    expect(order.status).toBe('PENDING');
    expect(order.totalAmount).toBe(59.98); // 2 * 29.99
  });

  test('should retrieve created order', async () => {
    // Create an order first
    const orderData: OrderRequest = {
      customerId: testCustomerId,
      items: [
        {
          productId: 'PROD-002',
          quantity: 1,
          price: 49.99,
        },
      ],
      shippingAddress: {
        street: '456 Park Ave',
        city: 'Chicago',
        state: 'IL',
        zip: '60601',
      },
    };

    const createdOrder = await apiGateway.createOrder(orderData);

    // Retrieve the order
    const retrievedOrder = await apiGateway.getOrder(createdOrder.orderId);

    // Verify order details
    expect(retrievedOrder.orderId).toBe(createdOrder.orderId);
    expect(retrievedOrder.status).toBe('PENDING');
    expect(retrievedOrder.items.length).toBe(1);
  });

  test('should update order status', async () => {
    // Create and then update order
    const orderData: OrderRequest = {
      customerId: testCustomerId,
      items: [
        {
          productId: 'PROD-003',
          quantity: 1,
          price: 99.99,
        },
      ],
      shippingAddress: {
        street: '789 Oak Ln',
        city: 'Aurora',
        state: 'IL',
        zip: '60504',
      },
    };

    const order = await apiGateway.createOrder(orderData);

    // Update order status to CONFIRMED
    const updatedOrder = await apiGateway.updateOrder(order.orderId, 'CONFIRMED');

    expect(updatedOrder.status).toBe('CONFIRMED');
  });

  test('should cancel order', async () => {
    // Create order
    const orderData: OrderRequest = {
      customerId: testCustomerId,
      items: [
        {
          productId: 'PROD-004',
          quantity: 1,
          price: 39.99,
        },
      ],
      shippingAddress: {
        street: '321 Elm St',
        city: 'Naperville',
        state: 'IL',
        zip: '60540',
      },
    };

    const order = await apiGateway.createOrder(orderData);

    // Cancel order
    await apiGateway.cancelOrder(order.orderId);

    // Verify order is cancelled
    const cancelledOrder = await apiGateway.getOrder(order.orderId);
    expect(cancelledOrder.status).toBe('CANCELLED');
  });

  test('should list all orders for customer', async () => {
    // Create multiple orders
    const orderData1: OrderRequest = {
      customerId: testCustomerId,
      items: [{ productId: 'PROD-005', quantity: 1, price: 25.00 }],
      shippingAddress: {
        street: '111 Pine St',
        city: 'Peoria',
        state: 'IL',
        zip: '61614',
      },
    };

    const orderData2: OrderRequest = {
      customerId: testCustomerId,
      items: [{ productId: 'PROD-006', quantity: 2, price: 50.00 }],
      shippingAddress: {
        street: '222 Maple Ave',
        city: 'Rockford',
        state: 'IL',
        zip: '61101',
      },
    };

    await apiGateway.createOrder(orderData1);
    await apiGateway.createOrder(orderData2);

    // List orders
    const orders = await apiGateway.listOrders(testCustomerId);

    expect(orders.length).toBeGreaterThanOrEqual(2);
    expect(orders.every((o) => o.customerId === testCustomerId)).toBe(true);
  });

  test('should fail creating order with invalid data', async ({ request }) => {
    const invalidOrder = {
      customerId: '', // Empty customer ID
      items: [], // Empty items
    };

    const response = await request.post('http://localhost:8080/api/v1/orders', {
      data: invalidOrder,
    });

    // Should return 400 Bad Request
    expect(response.status()).toBe(400);
  });

  test('should fail getting non-existent order', async ({ request }) => {
    const response = await request.get('http://localhost:8080/api/v1/orders/INVALID-ID');

    // Should return 404 Not Found
    expect(response.status()).toBe(404);
  });
});
```

### tests/cross-service-integration.spec.ts

```typescript
import { test, expect } from '../fixtures/auth.fixture';
import { ApiGatewayPage, OrderRequest } from '../pages/api-gateway.page';
import { InventoryPage } from '../pages/inventory.page';
import { v4 as uuidv4 } from 'uuid';

test.describe('Cross-Service Integration', () => {
  let apiGateway: ApiGatewayPage;
  let inventory: InventoryPage;
  let testCustomerId: string;

  test.beforeEach(async ({ request }) => {
    apiGateway = new ApiGatewayPage(request);
    inventory = new InventoryPage(request);
    testCustomerId = uuidv4();
  });

  test('should reserve inventory when order is created', async () => {
    const productId = 'PROD-INT-001';
    const quantity = 3;

    // Check initial stock
    const initialStock = await inventory.getAvailableStock(productId);

    // Create order (should trigger inventory reservation)
    const orderData: OrderRequest = {
      customerId: testCustomerId,
      items: [
        {
          productId,
          quantity,
          price: 49.99,
        },
      ],
      shippingAddress: {
        street: '999 Test St',
        city: 'TestCity',
        state: 'IL',
        zip: '12345',
      },
    };

    const order = await apiGateway.createOrder(orderData);

    // Wait for async inventory processing
    await new Promise((resolve) => setTimeout(resolve, 1000));

    // Verify stock was reserved
    const afterOrderStock = await inventory.getAvailableStock(productId);
    expect(afterOrderStock).toBe(initialStock - quantity);
  });

  test('should release inventory when order is cancelled', async () => {
    const productId = 'PROD-INT-002';
    const quantity = 2;

    const initialStock = await inventory.getAvailableStock(productId);

    // Create order
    const orderData: OrderRequest = {
      customerId: testCustomerId,
      items: [
        {
          productId,
          quantity,
          price: 69.99,
        },
      ],
      shippingAddress: {
        street: '888 Test Ave',
        city: 'TestVille',
        state: 'IL',
        zip: '54321',
      },
    };

    const order = await apiGateway.createOrder(orderData);
    await new Promise((resolve) => setTimeout(resolve, 500));

    const afterOrderStock = await inventory.getAvailableStock(productId);
    expect(afterOrderStock).toBe(initialStock - quantity);

    // Cancel order
    await apiGateway.cancelOrder(order.orderId);
    await new Promise((resolve) => setTimeout(resolve, 500));

    // Verify stock was released
    const afterCancellationStock = await inventory.getAvailableStock(productId);
    expect(afterCancellationStock).toBe(initialStock);
  });
});
```

## Step 6: Package.json Dependencies

```json
{
  "name": "e2e-tests",
  "version": "1.0.0",
  "description": "E2E tests for order management microservices",
  "scripts": {
    "test": "playwright test",
    "test:chromium": "playwright test --project=chromium",
    "test:firefox": "playwright test --project=firefox",
    "test:webkit": "playwright test --project=webkit",
    "test:debug": "playwright test --debug",
    "report": "playwright show-report",
    "ci": "playwright test --reporter=junit"
  },
  "devDependencies": {
    "@faker-js/faker": "^8.3.1",
    "@playwright/test": "^1.40.0",
    "@types/node": "^20.10.0",
    "typescript": "^5.3.3"
  },
  "dependencies": {
    "axios": "^1.6.2",
    "dotenv": "^16.3.1",
    "oracledb": "^6.2.0",
    "uuid": "^9.0.1"
  }
}
```

## Step 7: Running Tests

```bash
# Install dependencies
npm install

# Run all tests
npm test

# Run specific browser
npm run test:chromium

# Run with debug mode
npm run test:debug

# View HTML report
npm run report

# Run in CI mode
npm run ci
```

## .env File for Local Testing

```bash
# API Configuration
BASE_URL=http://localhost:8080
AUTH_SERVICE_URL=http://localhost:9000

# Database Configuration
DB_USERNAME=test_user
DB_PASSWORD=test_pwd
DB_CONNECT_STRING=localhost:1521/XEPDB1

# Test User
TEST_USERNAME=test_user
TEST_PASSWORD=test_password
```

## CI/CD Integration (GitHub Actions)

```yaml
name: E2E Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      oracle:
        image: container-registry.oracle.com/database/express:latest
        options: >-
          --health-cmd="sqlplus -v"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run ci
      - uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
```

## Success Criteria

✅ All test scenarios pass against microservices  
✅ Cross-service integration validated  
✅ 80%+ API endpoint coverage  
✅ Error scenarios covered  
✅ CI/CD pipeline green  
✅ HTML reports generated automatically  
