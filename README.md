# E-commerce Microservices Testing Documentation

This documentation provides a comprehensive guide for testing the complete workflow of the integrated E-commerce Microservices system with **RabbitMQ-based asynchronous order processing**. With our product-inventory integration and message-driven architecture, creating products automatically creates inventory records, and order processing now happens asynchronously for improved scalability and resilience.

## System Architecture

![alt text](./images/E-commerce%20Arch.svg)

## Key Features

- **Asynchronous Order Processing**: Orders are processed using RabbitMQ message queues for better scalability and fault tolerance
- **Automatic Inventory Management**: Creating products automatically creates corresponding inventory records
- **Event-Driven Architecture**: Services communicate through message queues, reducing tight coupling
- **Resilient Design**: System continues to function even if individual services experience temporary outages

## Run the Application

Run the application using docker compose:

```bash
docker-compose up --build -d
```

This will start all services including:
- Product Service (MongoDB)
- Order Service (MongoDB) 
- Inventory Service (PostgreSQL)
- User Service (PostgreSQL)
- RabbitMQ Message Broker
- Nginx API Gateway

## Install `jq`

Install `jq` to make JSON output more readable in the terminal:

```bash
apt-get update && apt-get install -y jq
```

## Overview of Testing Flow

1. User Registration and Authentication
2. Adding User Address
3. Creating Products (Inventory is automatically created)
4. Browsing Products and Checking Inventory
5. **Asynchronous Order Processing** (New RabbitMQ-based flow)
6. Viewing Order Details and Status Updates
7. **Message Queue Monitoring** (New)

Let's begin testing each component of the system.

## 1. User Registration and Authentication

First, we'll register a new user and obtain an authentication token.

### Step 1.1: Register a New User

```bash
curl -X POST "http://localhost/api/v1/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "customer@example.com",
    "password": "Password123",
    "first_name": "Example",
    "last_name": "Customer",
    "phone": "555-123-4567"
  }' | jq .
```

Expected response:
```json
{
  "id": 1,
  "email": "customer@example.com",
  "first_name": "Example",
  "last_name": "Customer",
  "phone": "555-123-4567",
  "is_active": true,
  "created_at": "2025-05-15T11:30:00.123456+00:00",
  "addresses": []
}
```

### Step 1.2: Login to Get Authentication Token

```bash
curl -X POST "http://localhost/api/v1/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=customer@example.com&password=Password123" | jq .
```

Expected response:
```json
{
  "access_token": "eyJhbGciOiJS...",
  "refresh_token": "eyJhbGciOiJS...",
  "token_type": "bearer"
}
```

Save the access_token for subsequent requests:

```bash
TOKEN="eyJhbGciOiJS..."  # Replace with the actual token from the response
```

### Step 1.3: Verify User Authentication

```bash
curl -X GET "http://localhost/api/v1/users/me" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
{
  "id": 1,
  "email": "customer@example.com",
  "first_name": "Example",
  "last_name": "Customer",
  "phone": "555-123-4567",
  "is_active": true,
  "created_at": "2025-05-15T11:30:00.123456+00:00",
  "addresses": []
}
```

## 2. Adding User Address

Next, we'll add a shipping address for the user.

```bash
curl -X POST "http://localhost/api/v1/users/me/addresses" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "line1": "123 Example Street",
    "line2": "Apt 4B",
    "city": "Example City",
    "state": "EX",
    "postal_code": "12345",
    "country": "Example Country",
    "is_default": true
  }' | jq .
```

Expected response:
```json
{
  "id": 1,
  "line1": "123 Example Street",
  "line2": "Apt 4B",
  "city": "Example City",
  "state": "EX",
  "postal_code": "12345",
  "country": "Example Country",
  "is_default": true
}
```

## 3. Creating Products with Automatic Inventory

Now, we'll create three different products. Because of our integration, inventory records will be automatically created for each product.

### Step 3.1: Create Product 1 - Smartphone

```bash
curl -X POST "http://localhost/api/v1/products/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Premium Smartphone",
    "description": "Latest model with high-end camera and long battery life",
    "category": "Electronics",
    "price": 899.99,
    "quantity": 50
  }' | jq .
```

Expected response (save the `_id` for later use):
```json
{
  "name": "Premium Smartphone",
  "description": "Latest model with high-end camera and long battery life",
  "category": "Electronics",
  "price": 899.99,
  "quantity": 50,
  "_id": "product_id_1"
}
```

```bash
PRODUCT_ID_1="product_id_1"  # Replace with the actual ID from the response
```

### Step 3.2: Create Product 2 - Wireless Headphones

```bash
curl -X POST "http://localhost/api/v1/products/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Wireless Noise-Cancelling Headphones",
    "description": "Premium headphones with active noise cancellation",
    "category": "Audio",
    "price": 249.99,
    "quantity": 100
  }' | jq .
```

Expected response:
```json
{
  "name": "Wireless Noise-Cancelling Headphones",
  "description": "Premium headphones with active noise cancellation",
  "category": "Audio",
  "price": 249.99,
  "quantity": 100,
  "_id": "product_id_2"
}
```

```bash
PRODUCT_ID_2="product_id_2"  # Replace with the actual ID from the response
```

### Step 3.3: Create Product 3 - Smart Watch

```bash
curl -X POST "http://localhost/api/v1/products/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Smart Fitness Watch",
    "description": "Waterproof fitness tracker with heart rate monitoring",
    "category": "Wearables",
    "price": 179.99,
    "quantity": 75
  }' | jq .
```

Expected response:
```json
{
  "name": "Smart Fitness Watch",
  "description": "Waterproof fitness tracker with heart rate monitoring",
  "category": "Wearables",
  "price": 179.99,
  "quantity": 75,
  "_id": "product_id_3"
}
```

```bash
PRODUCT_ID_3="product_id_3"  # Replace with the actual ID from the response
```

## 4. Browsing Products and Checking Inventory

Now, let's browse products and verify that inventory was automatically created.

### Step 4.1: Get All Products

```bash
curl -X GET "http://localhost/api/v1/products/" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
[
  {
    "name": "Premium Smartphone",
    "description": "Latest model with high-end camera and long battery life",
    "category": "Electronics",
    "price": 899.99,
    "quantity": 50,
    "_id": "product_id_1"
  },
  {
    "name": "Wireless Noise-Cancelling Headphones",
    "description": "Premium headphones with active noise cancellation",
    "category": "Audio",
    "price": 249.99,
    "quantity": 100,
    "_id": "product_id_2"
  },
  {
    "name": "Smart Fitness Watch",
    "description": "Waterproof fitness tracker with heart rate monitoring",
    "category": "Wearables",
    "price": 179.99,
    "quantity": 75,
    "_id": "product_id_3"
  }
]
```

### Step 4.2: Verify Inventory Was Created For Products

Check inventory for Product 1:

```bash
curl -X GET "http://localhost/api/v1/inventory/$PRODUCT_ID_1" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
{
  "product_id": "product_id_1",
  "available_quantity": 50,
  "reserved_quantity": 0,
  "reorder_threshold": 5,
  "id": 1,
  "created_at": "2025-05-15T11:40:00.123456+00:00",
  "updated_at": "2025-05-15T11:40:00.123456+00:00"
}
```

Check inventory for Product 2:

```bash
curl -X GET "http://localhost/api/v1/inventory/$PRODUCT_ID_2" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
{
  "product_id": "product_id_2",
  "available_quantity": 100,
  "reserved_quantity": 0,
  "reorder_threshold": 10,
  "id": 2,
  "created_at": "2025-05-15T11:41:00.123456+00:00",
  "updated_at": "2025-05-15T11:41:00.123456+00:00"
}
```

### Step 4.3: Filter Products by Category

```bash
curl -X GET "http://localhost/api/v1/products/?category=Electronics" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
[
  {
    "name": "Premium Smartphone",
    "description": "Latest model with high-end camera and long battery life",
    "category": "Electronics",
    "price": 899.99,
    "quantity": 50,
    "_id": "product_id_1"
  }
]
```

### Step 4.4: Filter Products by Price Range

```bash
curl -X GET "http://localhost/api/v1/products/?min_price=100&max_price=300" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
[
  {
    "name": "Wireless Noise-Cancelling Headphones",
    "description": "Premium headphones with active noise cancellation",
    "category": "Audio",
    "price": 249.99,
    "quantity": 100,
    "_id": "product_id_2"
  },
  {
    "name": "Smart Fitness Watch",
    "description": "Waterproof fitness tracker with heart rate monitoring",
    "category": "Wearables",
    "price": 179.99,
    "quantity": 75,
    "_id": "product_id_3"
  }
]
```

## 5. Asynchronous Order Processing (NEW)

With RabbitMQ integration, order processing is now asynchronous and more resilient. Here's how the new flow works:

### Order Processing Flow

1. **Order Creation**: Customer places order → Order Service creates pending order
2. **Message Publishing**: Order Service publishes order to `order_created` queue
3. **Inventory Processing**: Inventory Service processes order and reserves inventory
4. **Status Updates**: Order status is updated based on inventory availability
5. **Completion**: Order progresses through processing stages asynchronously

### Step 5.1: Get User ID

First, we need to get the user's ID:

```bash
curl -X GET "http://localhost/api/v1/users/me" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:

```json
{
  "email": "customer@example.com",
  "first_name": "Example",
  "last_name": "Customer",
  "phone": "555-123-4567",
  "id": 1,
  "is_active": true,
  "created_at": "2025-05-15T14:56:27.118728+00:00",
  "addresses": [
    {
      "line1": "123 Example Street",
      "line2": "Apt 4B",
      "city": "Example City",
      "state": "EX",
      "postal_code": "12345",
      "country": "Example Country",
      "is_default": true,
      "id": 1
    }
  ]
}
```

From the response, save the user ID:

```bash
USER_ID="1"  # Replace with the actual ID from the response
```

### Step 5.2: Place an Order for a Single Product (Asynchronous Processing)

```bash
curl -X POST "http://localhost/api/v1/orders/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "'$USER_ID'",
    "items": [
      {
        "product_id": "'$PRODUCT_ID_1'",
        "quantity": 1,
        "price": 899.99
      }
    ],
    "shipping_address": {
      "line1": "123 Example Street",
      "line2": "Apt 4B",
      "city": "Example City",
      "state": "EX",
      "postal_code": "12345",
      "country": "Example Country"
    }
  }' | jq .
```

Expected response (save the `_id` for later use):
```json
{
  "_id": "order_id_1",
  "user_id": "1",
  "items": [
    {
      "product_id": "product_id_1",
      "quantity": 1,
      "price": 899.99
    }
  ],
  "total_price": 899.99,
  "status": "pending",
  "shipping_address": {
    "line1": "123 Example Street",
    "line2": "Apt 4B",
    "city": "Example City",
    "state": "EX",
    "postal_code": "12345",
    "country": "Example Country"
  },
  "created_at": "2025-05-15T11:50:00.123456+00:00",
  "updated_at": "2025-05-15T11:50:00.123456+00:00"
}
```

```bash
ORDER_ID_1="order_id_1"  # Replace with the actual ID from the response
```

**Note**: The order is initially created in "pending" status. The asynchronous processing will update its status based on inventory availability.

### Step 5.3: Monitor Order Status Changes

Wait a few seconds for the asynchronous processing to complete, then check the order status:

```bash
# Wait for processing
sleep 5

# Check updated order status
curl -X GET "http://localhost/api/v1/orders/$ORDER_ID_1" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

You should see the order status has changed from "pending" to "processing" or "paid" (depending on implementation).

### Step 5.4: Verify Inventory Was Reserved

Check that inventory was properly reserved:

```bash
curl -X GET "http://localhost/api/v1/inventory/$PRODUCT_ID_1" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response showing reserved inventory:
```json
{
  "product_id": "product_id_1",
  "available_quantity": 49,
  "reserved_quantity": 1,
  "reorder_threshold": 5,
  "id": 1,
  "created_at": "2025-05-15T11:40:00.123456+00:00",
  "updated_at": "2025-05-15T11:50:00.123456+00:00"
}
```

### Step 5.5: Place an Order for Multiple Products

```bash
curl -X POST "http://localhost/api/v1/orders/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "'$USER_ID'",
    "items": [
      {
        "product_id": "'$PRODUCT_ID_2'",
        "quantity": 1,
        "price": 249.99
      },
      {
        "product_id": "'$PRODUCT_ID_3'",
        "quantity": 2,
        "price": 179.99
      }
    ],
    "shipping_address": {
      "line1": "123 Example Street",
      "line2": "Apt 4B",
      "city": "Example City",
      "state": "EX",
      "postal_code": "12345",
      "country": "Example Country"
    }
  }' | jq .
```

Expected response:
```json
{
  "_id": "order_id_2",
  "user_id": "1",
  "items": [
    {
      "product_id": "product_id_2",
      "quantity": 1,
      "price": 249.99
    },
    {
      "product_id": "product_id_3",
      "quantity": 2,
      "price": 179.99
    }
  ],
  "total_price": 609.97,
  "status": "pending",
  "shipping_address": {
    "line1": "123 Example Street",
    "line2": "Apt 4B",
    "city": "Example City",
    "state": "EX",
    "postal_code": "12345",
    "country": "Example Country"
  },
  "created_at": "2025-05-15T11:55:00.123456+00:00",
  "updated_at": "2025-05-15T11:55:00.123456+00:00"
}
```

```bash
ORDER_ID_2="order_id_2"  # Replace with the actual ID from the response
```

## 6. Viewing and Managing Orders

Now, let's view and manage the orders.

### Step 6.1: Get Order Details

```bash
curl -X GET "http://localhost/api/v1/orders/$ORDER_ID_1" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
{
  "_id": "order_id_1",
  "user_id": "1",
  "items": [
    {
      "product_id": "product_id_1",
      "quantity": 1,
      "price": 899.99
    }
  ],
  "total_price": 899.99,
  "status": "paid",
  "shipping_address": {
    "line1": "123 Example Street",
    "line2": "Apt 4B",
    "city": "Example City",
    "state": "EX",
    "postal_code": "12345",
    "country": "Example Country"
  },
  "created_at": "2025-05-15T11:50:00.123456+00:00",
  "updated_at": "2025-05-15T11:50:00.123456+00:00"
}
```

### Step 6.2: List All Orders of the User

```bash
curl -X GET "http://localhost/api/v1/orders/" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
[
  {
    "_id": "order_id_2",
    "user_id": "1",
    "items": [
      {
        "product_id": "product_id_2",
        "quantity": 1,
        "price": 249.99
      },
      {
        "product_id": "product_id_3",
        "quantity": 2,
        "price": 179.99
      }
    ],
    "total_price": 609.97,
    "status": "paid",
    "shipping_address": {
      "line1": "123 Example Street",
      "line2": "Apt 4B",
      "city": "Example City",
      "state": "EX",
      "postal_code": "12345",
      "country": "Example Country"
    },
    "created_at": "2025-05-15T11:55:00.123456+00:00",
    "updated_at": "2025-05-15T11:55:00.123456+00:00"
  },
  {
    "_id": "order_id_1",
    "user_id": "1",
    "items": [
      {
        "product_id": "product_id_1",
        "quantity": 1,
        "price": 899.99
      }
    ],
    "total_price": 899.99,
    "status": "paid",
    "shipping_address": {
      "line1": "123 Example Street",
      "line2": "Apt 4B",
      "city": "Example City",
      "state": "EX",
      "postal_code": "12345",
      "country": "Example Country"
    },
    "created_at": "2025-05-15T11:50:00.123456+00:00",
    "updated_at": "2025-05-15T11:50:00.123456+00:00"
  }
]
```

### Step 6.3: Update Order Status

```bash
curl -X PUT "http://localhost/api/v1/orders/$ORDER_ID_1/status" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "shipped"
  }' | jq .
```

Expected response:
```json
{
  "_id": "order_id_1",
  "user_id": "1",
  "items": [
    {
      "product_id": "product_id_1",
      "quantity": 1,
      "price": 899.99
    }
  ],
  "total_price": 899.99,
  "status": "shipped",
  "shipping_address": {
    "line1": "123 Example Street",
    "line2": "Apt 4B",
    "city": "Example City",
    "state": "EX",
    "postal_code": "12345",
    "country": "Example Country"
  },
  "created_at": "2025-05-15T11:50:00.123456+00:00",
  "updated_at": "2025-05-15T12:00:00.123456+00:00"
}
```

### Step 6.4: Cancel an Order (Asynchronous Inventory Release)

```bash
curl -X DELETE "http://localhost/api/v1/orders/$ORDER_ID_2" \
  -H "Authorization: Bearer $TOKEN"
```

This should return a 204 No Content status with no response body. Now let's verify the order was cancelled:

```bash
curl -X GET "http://localhost/api/v1/orders/$ORDER_ID_2" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
{
  "_id": "order_id_2",
  "user_id": "1",
  "items": [
    {
      "product_id": "product_id_2",
      "quantity": 1,
      "price": 249.99
    },
    {
      "product_id": "product_id_3",
      "quantity": 2,
      "price": 179.99
    }
  ],
  "total_price": 609.97,
  "status": "cancelled",
  "shipping_address": {
    "line1": "123 Example Street",
    "line2": "Apt 4B",
    "city": "Example City",
    "state": "EX",
    "postal_code": "12345",
    "country": "Example Country"
  },
  "created_at": "2025-05-15T11:55:00.123456+00:00",
  "updated_at": "2025-05-15T12:05:00.123456+00:00"
}
```

### Step 6.5: Check Inventory After Order Operations

Let's check the inventory for Product 1 after placing an order:

```bash
curl -X GET "http://localhost/api/v1/inventory/$PRODUCT_ID_1" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
{
  "product_id": "product_id_1",
  "available_quantity": 49,
  "reserved_quantity": 1,
  "reorder_threshold": 5,
  "id": 1,
  "created_at": "2025-05-15T11:40:00.123456+00:00",
  "updated_at": "2025-05-15T11:50:00.123456+00:00"
}
```

And check inventory for Product 2 after cancelling an order (inventory should be released):

```bash
# Wait for async processing
sleep 5

curl -X GET "http://localhost/api/v1/inventory/$PRODUCT_ID_2" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

Expected response:
```json
{
  "product_id": "product_id_2",
  "available_quantity": 100,
  "reserved_quantity": 0,
  "reorder_threshold": 10,
  "id": 2,
  "created_at": "2025-05-15T11:41:00.123456+00:00",
  "updated_at": "2025-05-15T12:05:00.123456+00:00"
}
```

## 7. Monitoring RabbitMQ Message Queues (NEW)

### Step 7.1: Access RabbitMQ Management Interface

Open your browser and navigate to:
```
http://localhost:15672/
```

Login credentials:
- Username: `guest`
- Password: `guest`

### Step 7.2: Monitor Queues

In the RabbitMQ management interface, you can monitor:

- **Queue Overview**: See all active queues and their message counts
- **Message Rates**: Monitor message publishing and consumption rates
- **Connections**: View active connections from each service
- **Exchanges**: See message routing information

Key queues to monitor:
- `order_created`: New orders waiting for inventory processing
- `inventory_reserved`: Orders with successfully reserved inventory
- `inventory_failed`: Orders where inventory reservation failed
- `order_processed`: Completed orders ready for fulfillment
- `inventory_release`: Inventory release requests from cancelled orders

### Step 7.3: Queue Status via API

You can also check queue status programmatically:

```bash
# Check all queues
curl -s -u guest:guest http://localhost:15672/api/queues | jq '.[] | {name: .name, messages: .messages, consumers: .consumers}'

# Check specific queue
curl -s -u guest:guest http://localhost:15672/api/queues/%2F/order_created | jq '{name: .name, messages: .messages}'
```

## 8. System Health Checks

### Step 8.1: Check All Services

```bash
# Check service status
docker-compose ps

# Check individual service health
curl -X GET "http://localhost/health"
curl -X GET "http://localhost:8000/health"  # Product Service
curl -X GET "http://localhost:8001/health"  # Order Service  
curl -X GET "http://localhost:8002/health"  # Inventory Service
curl -X GET "http://localhost:8003/health"  # User Service
```

### Step 8.2: Check Service Logs

```bash
# Check for any errors in service logs
docker-compose logs --tail=20 product-service
docker-compose logs --tail=20 order-service
docker-compose logs --tail=20 inventory-service
docker-compose logs --tail=20 user-service
docker-compose logs --tail=20 rabbitmq
```

## 9. Testing System Resilience (NEW)

### Step 9.1: Test Service Outage Resilience

```bash
# Stop inventory service
docker-compose stop inventory-service

# Place an order (should be accepted but stay in pending state)
curl -X POST "http://localhost/api/v1/orders/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "'$USER_ID'",
    "items": [
      {
        "product_id": "'$PRODUCT_ID_1'",
        "quantity": 1,
        "price": 899.99
      }
    ],
    "shipping_address": {
      "line1": "123 Example Street",
      "line2": "Apt 4B",
      "city": "Example City",
      "state": "EX",
      "postal_code": "12345",
      "country": "Example Country"
    }
  }' | jq .

# Save the order ID
RESILIENCE_ORDER_ID="order_id_3"  # Replace with actual ID

# Restart inventory service
docker-compose start inventory-service

# Wait and check if order was processed
sleep 30
curl -X GET "http://localhost/api/v1/orders/$RESILIENCE_ORDER_ID" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

The order should eventually transition from "pending" to "paid" status after the inventory service comes back online, demonstrating system resilience.

## 10. Verifying the Complete Asynchronous Workflow

To verify that the complete e-commerce flow works end-to-end with our integrated product-inventory system and RabbitMQ-based asynchronous processing, we've successfully:

1. ✅ Registered a user account and authenticated
2. ✅ Added a shipping address
3. ✅ Created three different products with automatic inventory creation
4. ✅ Verified that inventory was created for each product
5. ✅ Browsed and filtered products
6. ✅ **Placed orders using asynchronous message-based processing**
7. ✅ **Monitored message queues and processing status**
8. ✅ Viewed order details and status updates
9. ✅ Updated order status
10. ✅ Cancelled an order with automatic inventory release
11. ✅ **Verified inventory changes after asynchronous order operations**
12. ✅ **Tested system resilience during service outages**

## Key Improvements with RabbitMQ Integration

- **Scalability**: Asynchronous processing allows the system to handle high order volumes without blocking
- **Resilience**: Services can temporarily go offline without losing orders - messages wait in queues
- **Decoupling**: Services are no longer tightly coupled, improving maintainability and deployment flexibility
- **Load Leveling**: Message queues absorb traffic spikes, preventing service overload
- **Event-Driven Architecture**: Services react to events rather than direct API calls, improving responsiveness
- **Fault Tolerance**: Failed message processing can be retried or sent to dead letter queues
- **Monitoring**: Message queue metrics provide insights into system performance and bottlenecks

## Architecture Components

### Microservices
- **User Service** (Port 8003): Authentication, user management, addresses
- **Product Service** (Port 8000): Product catalog and inventory integration
- **Inventory Service** (Port 8002): Stock management and reservation
- **Order Service** (Port 8001): Order processing and workflow orchestration

### Infrastructure
- **RabbitMQ** (Ports 5672/15672): Message broker for asynchronous communication
- **Nginx** (Port 80): API Gateway and load balancer
- **MongoDB**: Document storage for products and orders
- **PostgreSQL**: Relational storage for users and inventory

### Message Queues
- `order_created`: New orders awaiting inventory processing
- `inventory_reserved`: Orders with successful inventory reservation
- `inventory_failed`: Orders with inventory reservation failures
- `order_processed`: Orders ready for fulfillment
- `inventory_release`: Inventory release requests from cancellations

## Environment Configuration

Each service requires specific environment variables:

### RabbitMQ Configuration (All Services)
```bash
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
```

### Queue Names
```bash
ORDER_CREATED_QUEUE=order_created
INVENTORY_RESERVED_QUEUE=inventory_reserved
INVENTORY_FAILED_QUEUE=inventory_failed
ORDER_PROCESSED_QUEUE=order_processed
```

## Troubleshooting

### Common Issues

1. **502 Bad Gateway Errors**
   - Check service health endpoints
   - Verify Nginx configuration paths
   - Ensure services are on the same Docker network

2. **RabbitMQ Connection Failures**
   - Verify RabbitMQ is running: `docker-compose ps rabbitmq`
   - Check connection strings in service configuration
   - Monitor RabbitMQ logs: `docker-compose logs rabbitmq`

3. **Orders Stuck in Pending Status**
   - Check inventory service logs for processing errors
   - Verify message queue activity in RabbitMQ management interface
   - Ensure sufficient inventory is available

4. **Service Import Errors**
   - Verify all required dependencies are in requirements.txt
   - Check that messaging directories exist in each service
   - Ensure proper import statements in route files

### Debugging Commands

```bash
# Check all service status
docker-compose ps

# View service logs
docker-compose logs [service-name] --tail=50

# Check RabbitMQ queues
curl -s -u guest:guest http://localhost:15672/api/queues

# Test direct service communication
docker-compose exec nginx-gateway curl http://[service-name]:[port]/health

# Monitor real-time logs
docker-compose logs -f [service-name]
```

## Performance Testing

### Load Testing Script

```bash
#!/bin/bash
# load_test.sh - Simple load testing script

TOKEN="your_token_here"
PRODUCT_ID="your_product_id_here"
USER_ID="your_user_id_here"

echo "Starting load test with 20 concurrent orders..."

for i in {1..20}; do
  {
    echo "Placing order $i..."
    curl -s -X POST "http://localhost/api/v1/orders/" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d '{
        "user_id": "'$USER_ID'",
        "items": [
          {
            "product_id": "'$PRODUCT_ID'",
            "quantity": 1,
            "price": 19.99
          }
        ],
        "shipping_address": {
          "line1": "123 Test St",
          "city": "Test City",
          "state": "TS",
          "postal_code": "12345",
          "country": "Test Country"
        }
      }' > /dev/null
    echo "Order $i completed."
  } &
done

wait
echo "Load test completed!"
```

Usage:
```bash
chmod +x load_test.sh
./load_test.sh
```

## Production Considerations

### Security
- Replace default RabbitMQ credentials
- Implement proper JWT token validation
- Use HTTPS for all external communications
- Set up proper firewall rules

### Monitoring
- Implement health check endpoints for all services
- Set up logging aggregation (ELK stack or similar)
- Monitor message queue metrics and alerts
- Track order processing times and success rates

### Scalability
- Use horizontal pod autoscaling for high-traffic services
- Implement database connection pooling
- Consider message queue clustering for high availability
- Set up proper load balancing

### Data Persistence
- Configure proper database backups
- Use persistent volumes for RabbitMQ data
- Implement database migration strategies
- Plan for disaster recovery

## API Documentation

When all services are running, you can access Swagger documentation at:

- **Product Service**: http://localhost:8000/api/v1/docs
- **Order Service**: http://localhost:8001/api/v1/docs  
- **Inventory Service**: http://localhost:8002/api/v1/docs
- **User Service**: http://localhost:8003/api/v1/docs

## Next Steps

To further enhance the system, consider implementing:

1. **Payment Processing Service**: Handle actual payment transactions
2. **Shipping Service**: Manage shipping providers and tracking
3. **Notification Service**: Send email/SMS updates to customers
4. **Analytics Service**: Track user behavior and system metrics
5. **Recommendation Engine**: Suggest products based on user history
6. **Admin Dashboard**: Manage products, orders, and users
7. **Mobile API**: Optimize APIs for mobile applications
8. **Search Service**: Implement full-text search capabilities
9. **Caching Layer**: Add Redis for improved performance
10. **API Rate Limiting**: Implement request throttling

This completes the comprehensive testing and documentation for the RabbitMQ-integrated e-commerce microservices system. The asynchronous architecture provides a robust foundation for scaling and handling real-world e-commerce loads while maintaining system reliability and performance.