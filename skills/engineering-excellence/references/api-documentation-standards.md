# API Naming, Documentation & Service Contracts

Covers: REST URL naming rules, Swagger/OpenAPI documentation, data contracts
between services, self-deployable services, infrastructure as code.

---

## 10. API & URL Naming Standards

Follow REST resource naming conventions. URLs are public contracts — inconsistency breaks clients.

```
# ❌ Bad — verbs, capital letters, camelCase, inconsistent patterns
GET  /GetUsers
POST /createOrder
GET  /userOrders/getUserById?Id=123
GET  /ProductCategories

# ✅ Good — nouns, lowercase, kebab-case, hierarchical
GET  /users
POST /orders
GET  /users/{id}/orders
GET  /product-categories
```

| Rule | Standard |
|---|---|
| Always lowercase | `/users` not `/Users` |
| Use hyphens `-` not underscores `_` | `/product-categories` not `/product_categories` |
| Plural nouns | `/orders` not `/order` |
| No verbs in URL | `/orders/{id}/cancel` (noun + action via POST) not `/cancelOrder` |
| Max 2 levels of nesting | `/users/{id}/orders` — avoid `/users/{id}/orders/{id}/items/{id}/reviews` |
| Query params for filtering | `/orders?status=pending&sort=created_at` |
| Report/file names: camelCase | `salesReport_2024Q1.csv` — camelCase for file names is acceptable |

**References:**
- https://restfulapi.net/resource-naming/
- https://moz.com/learn/seo/url
- https://medium.com/@victor.leong.17/5-url-best-practices-936329ba36a7

---

## 11. Swagger / API Documentation

Every service must have API documentation before release. Undocumented APIs are unmaintainable.

```yaml
# ✅ Good — NestJS Swagger decoration example
@ApiTags('orders')
@ApiBearerAuth()
@Controller('orders')
export class OrdersController {

  @Post()
  @ApiOperation({ summary: 'Create a new order' })
  @ApiResponse({ status: 201, description: 'Order created.', type: OrderResponse })
  @ApiResponse({ status: 422, description: 'Validation error.' })
  @ApiResponse({ status: 401, description: 'Unauthorized.' })
  async createOrder(@Body() dto: CreateOrderDto): Promise<OrderResponse> {
    return this.orderService.create(dto);
  }
}
```

```yaml
# ✅ Good — Serverless (Lambda) swagger in serverless.yml
functions:
  createOrder:
    handler: src/handlers/order.create
    events:
      - http:
          path: /orders
          method: post
          documentation:
            summary: Create a new order
            requestBody:
              description: Order creation payload
            requestModels:
              application/json: CreateOrderRequest
            methodResponses:
              - statusCode: 201
                responseBody:
                  description: Created order
```

**Also maintain:**
- Postman collection exported and committed to `docs/postman/`
- Stoplight.io or Swagger UI accessible in dev/staging environments
- Architecture diagram updated in `docs/architecture/` on every infrastructure change

---

## 12. Data Contracts Between Services

Every communication boundary between two services, components, or parties must have an
explicit, versioned data contract. This prevents misunderstandings and enables independent deployment.

```typescript
// ✅ Good — explicit contract as a shared DTO/interface
// contracts/order-created.event.ts  (shared between OrderService and InventoryService)

/**
 * OrderCreatedEvent — published by OrdersService, consumed by InventoryService.
 * Version: 1.0
 * Breaking change policy: create v2 event type; maintain v1 until all consumers migrated.
 */
export interface OrderCreatedEvent {
  eventId:       string;        // unique event identifier for deduplication
  version:       '1.0';         // event schema version
  timestamp:     string;        // ISO 8601 UTC
  orderId:       string;
  userId:        string;
  items: Array<{
    sku:        string;
    quantity:   number;
    unitPrice:  number;
  }>;
}
```

```typescript
// ✅ Good — API contract as an OpenAPI-aligned response DTO
// contracts/user-profile.response.ts
export class UserProfileResponse {
  @ApiProperty({ example: 'usr_abc123' })
  id: string;

  @ApiProperty({ example: 'John' })
  firstName: string;

  @ApiProperty({ example: 'Doe' })
  lastName: string;

  // NOTE: email deliberately omitted — not required by consuming clients
  // NOTE: passwordHash NEVER included
}
```

---

## 13. Self-Deployable Services & Infrastructure as Code

Each service must be independently deployable. Dependencies must be created as part of deployment.
No manual steps, no shared mutable infrastructure that requires coordination to deploy.

```yaml
# ✅ Good — serverless.yml creates all dependent resources on deploy
service: orders-service

provider:
  name: aws
  runtime: nodejs20.x
  environment:
    ORDERS_TABLE: !Ref OrdersTable
    EVENTS_QUEUE_URL: !Ref OrderEventsQueue

resources:
  Resources:
    OrdersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:service}-${sls:stage}-orders
        # ... table definition

    OrderEventsQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:service}-${sls:stage}-events
```

```hcl
# ✅ Good — Terraform creates all service dependencies atomically
module "orders_service" {
  source = "./modules/service"

  name        = "orders-service"
  environment = var.environment

  # All resources the service needs are defined here
  # No manual console clicks required
  database_instance_class = "db.t3.medium"
  queue_visibility_timeout = 300
}
```

---

## Gap Detection Table (Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| Verb in URL | `/getUsers`, `/createOrder`, `/deleteUser` | 🟡 |
| Uppercase in URL | `/Users`, `/ProductCategories` | 🟡 |
| Underscore in URL path | `/product_categories`, `/user_orders` | 🟡 |
| No Swagger/API documentation | Controller with no `@ApiOperation` or equivalent | 🟠 |
| No Postman collection | No `docs/postman/` or equivalent committed | 🟡 |
| Architecture diagram not updated | Infra change with no diagram update | 🟡 |
| No data contract for cross-service events | Event published with no versioned schema definition | 🟠 |
| Service not self-deployable | Deployment requires manual steps or shared mutable infra | 🟠 |
| Master dataset not maintained | Reference/lookup data hardcoded in code instead of DB/config | 🟡 |
