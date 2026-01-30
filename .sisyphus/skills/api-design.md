# API Design Standards

## General Principles

APIs are products, not implementations. Design with consumers in mind.

## REST APIs

### URL Patterns

```
Collection:    /v1/resources
Resource:      /v1/resources/{id}
Sub-resource:  /v1/resources/{id}/sub-resources
Query:         /v1/resources?param=value
```

### HTTP Status Codes

- 200: Success
- 201: Created
- 204: No Content
- 400: Bad Request
- 401: Unauthorized
- 403: Forbidden
- 404: Not Found
- 409: Conflict
- 429: Too Many Requests
- 500: Internal Server Error

### Requirements

- Resource-oriented URLs
- Proper HTTP method usage
- Consistent pagination
- Hypermedia links where appropriate

## GraphQL

### Schema Pattern

```graphql
type Query {
  resource(id: ID!): Resource
  resources(filter: ResourceFilter, pagination: PaginationInput): ResourceConnection!
}

type ResourceConnection {
  edges: [ResourceEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ResourceEdge {
  node: Resource!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### Requirements

- Schema-first design
- Focused queries with selection sets
- Mutations follow command pattern
- Clear error handling

## gRPC

### Service Pattern

```protobuf
service ResourceService {
  rpc GetResource(GetResourceRequest) returns (Resource);
  rpc ListResources(ListResourcesRequest) returns (stream Resource);
  rpc CreateResources(stream CreateResourceRequest) returns (CreateResourcesResponse);
  rpc ProcessResourceStream(stream ResourceOperation) returns (stream ResourceOperationResult);
}
```

### Requirements

- Service definitions with clear RPCs
- Consistent error codes
- Streaming for large data transfers
- Binary efficiency
