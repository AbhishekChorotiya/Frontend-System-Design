# REST vs GraphQL

## Table of Contents
- [Introduction](#introduction)
- [What is REST?](#what-is-rest)
- [What is GraphQL?](#what-is-graphql)
- [Key Differences](#key-differences)
- [REST Implementation](#rest-implementation)
- [GraphQL Implementation](#graphql-implementation)
- [Performance Comparison](#performance-comparison)
- [Use Cases](#use-cases)
- [Pros and Cons](#pros-and-cons)
- [Migration Strategies](#migration-strategies)
- [Common Beginner Doubts](#common-beginner-doubts)
- [Best Practices](#best-practices)
- [Conclusion](#conclusion)

## Introduction

When building frontend applications, one of the most critical decisions is how to handle data communication between the client and server. Two dominant approaches have emerged: REST (Representational State Transfer) and GraphQL. Each has its own philosophy, strengths, and ideal use cases. Understanding the differences between these approaches is crucial for making informed architectural decisions in frontend system design.

## What is REST?

REST (Representational State Transfer) is an architectural style for designing networked applications, particularly web services. It was introduced by Roy Fielding in 2000 and has become the de facto standard for web APIs.

### Core Principles of REST

1. **Stateless**: Each request contains all information needed to process it
2. **Client-Server Architecture**: Clear separation of concerns
3. **Cacheable**: Responses should be cacheable when appropriate
4. **Uniform Interface**: Consistent way to interact with resources
5. **Layered System**: Architecture can be composed of hierarchical layers
6. **Code on Demand** (optional): Server can send executable code to client

### REST Resource Structure

```javascript
// Example REST API endpoints
GET    /api/users           // Get all users
GET    /api/users/123       // Get user with ID 123
POST   /api/users           // Create a new user
PUT    /api/users/123       // Update user with ID 123
DELETE /api/users/123       // Delete user with ID 123

GET    /api/users/123/posts // Get posts by user 123
GET    /api/posts/456       // Get specific post
```

## What is GraphQL?

GraphQL is a query language and runtime for APIs developed by Facebook in 2012 and open-sourced in 2015. Unlike REST, which exposes multiple endpoints, GraphQL provides a single endpoint that allows clients to request exactly the data they need.

### Core Concepts of GraphQL

1. **Single Endpoint**: All requests go through one URL
2. **Strongly Typed**: Schema defines the structure of available data
3. **Client-Specified Queries**: Clients request exactly what they need
4. **Hierarchical**: Queries mirror the structure of returned data
5. **Introspective**: Schema is queryable and self-documenting

### GraphQL Schema Example

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  createdAt: String!
}

type Query {
  user(id: ID!): User
  users: [User!]!
  post(id: ID!): Post
  posts: [Post!]!
}

type Mutation {
  createUser(name: String!, email: String!): User!
  updateUser(id: ID!, name: String, email: String): User!
  deleteUser(id: ID!): Boolean!
}
```

## Key Differences

| Aspect | REST | GraphQL |
|--------|------|---------|
| **Endpoints** | Multiple endpoints | Single endpoint |
| **Data Fetching** | Fixed data structure | Flexible, client-specified |
| **Over/Under-fetching** | Common issue | Eliminated |
| **Caching** | HTTP caching works well | More complex caching |
| **Learning Curve** | Lower | Higher |
| **Tooling** | Mature ecosystem | Growing ecosystem |
| **File Uploads** | Native support | Requires additional specification |
| **Real-time** | Requires additional protocols | Built-in subscriptions |

## REST Implementation

### Frontend REST Implementation with Fetch API

```javascript
// REST API service class
class UserService {
  constructor(baseURL = '/api') {
    this.baseURL = baseURL;
  }

  async getAllUsers() {
    try {
      const response = await fetch(`${this.baseURL}/users`);
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      return await response.json();
    } catch (error) {
      console.error('Error fetching users:', error);
      throw error;
    }
  }

  async getUserById(id) {
    try {
      const response = await fetch(`${this.baseURL}/users/${id}`);
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      return await response.json();
    } catch (error) {
      console.error('Error fetching user:', error);
      throw error;
    }
  }

  async createUser(userData) {
    try {
      const response = await fetch(`${this.baseURL}/users`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(userData),
      });
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      return await response.json();
    } catch (error) {
      console.error('Error creating user:', error);
      throw error;
    }
  }

  async updateUser(id, userData) {
    try {
      const response = await fetch(`${this.baseURL}/users/${id}`, {
        method: 'PUT',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(userData),
      });
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      return await response.json();
    } catch (error) {
      console.error('Error updating user:', error);
      throw error;
    }
  }

  async deleteUser(id) {
    try {
      const response = await fetch(`${this.baseURL}/users/${id}`, {
        method: 'DELETE',
      });
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      return response.status === 204;
    } catch (error) {
      console.error('Error deleting user:', error);
      throw error;
    }
  }
}

// Usage in React component
import React, { useState, useEffect } from 'react';

const UserList = () => {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const userService = new UserService();

  useEffect(() => {
    const fetchUsers = async () => {
      try {
        setLoading(true);
        const userData = await userService.getAllUsers();
        setUsers(userData);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchUsers();
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <h2>Users</h2>
      <ul>
        {users.map(user => (
          <li key={user.id}>
            {user.name} - {user.email}
          </li>
        ))}
      </ul>
    </div>
  );
};

export default UserList;
```

### REST with Axios and Custom Hooks

```javascript
// Custom hook for REST API calls
import { useState, useEffect } from 'react';
import axios from 'axios';

// Configure axios instance
const api = axios.create({
  baseURL: '/api',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor for authentication
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor for error handling
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Handle unauthorized access
      localStorage.removeItem('authToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

// Custom hook for fetching data
export const useFetch = (url, dependencies = []) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true);
        setError(null);
        const response = await api.get(url);
        setData(response.data);
      } catch (err) {
        setError(err.response?.data?.message || err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, dependencies);

  return { data, loading, error };
};

// Custom hook for mutations
export const useMutation = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const mutate = async (method, url, data = null) => {
    try {
      setLoading(true);
      setError(null);
      const response = await api[method](url, data);
      return response.data;
    } catch (err) {
      setError(err.response?.data?.message || err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  return { mutate, loading, error };
};

// Usage in component
const UserProfile = ({ userId }) => {
  const { data: user, loading, error } = useFetch(`/users/${userId}`, [userId]);
  const { mutate, loading: updating } = useMutation();

  const handleUpdateUser = async (updatedData) => {
    try {
      await mutate('put', `/users/${userId}`, updatedData);
      // Refresh data or update local state
    } catch (error) {
      console.error('Failed to update user:', error);
    }
  };

  if (loading) return <div>Loading user...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      {/* Update form would go here */}
    </div>
  );
};
```

## GraphQL Implementation

### Frontend GraphQL Implementation with Apollo Client

```javascript
// Apollo Client setup
import { ApolloClient, InMemoryCache, createHttpLink, from } from '@apollo/client';
import { setContext } from '@apollo/client/link/context';
import { onError } from '@apollo/client/link/error';

// HTTP link
const httpLink = createHttpLink({
  uri: '/graphql',
});

// Auth link
const authLink = setContext((_, { headers }) => {
  const token = localStorage.getItem('authToken');
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : "",
    }
  };
});

// Error link
const errorLink = onError(({ graphQLErrors, networkError, operation, forward }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, locations, path }) =>
      console.error(`GraphQL error: Message: ${message}, Location: ${locations}, Path: ${path}`)
    );
  }

  if (networkError) {
    console.error(`Network error: ${networkError}`);
    if (networkError.statusCode === 401) {
      localStorage.removeItem('authToken');
      window.location.href = '/login';
    }
  }
});

// Apollo Client instance
const client = new ApolloClient({
  link: from([errorLink, authLink, httpLink]),
  cache: new InMemoryCache({
    typePolicies: {
      User: {
        fields: {
          posts: {
            merge(existing = [], incoming) {
              return [...existing, ...incoming];
            },
          },
        },
      },
    },
  }),
  defaultOptions: {
    watchQuery: {
      errorPolicy: 'all',
    },
    query: {
      errorPolicy: 'all',
    },
  },
});

export default client;
```

### GraphQL Queries and Mutations

```javascript
import { gql, useQuery, useMutation } from '@apollo/client';

// GraphQL queries
const GET_USERS = gql`
  query GetUsers {
    users {
      id
      name
      email
      posts {
        id
        title
        createdAt
      }
    }
  }
`;

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
      posts {
        id
        title
        content
        createdAt
      }
    }
  }
`;

// GraphQL mutations
const CREATE_USER = gql`
  mutation CreateUser($name: String!, $email: String!) {
    createUser(name: $name, email: $email) {
      id
      name
      email
    }
  }
`;

const UPDATE_USER = gql`
  mutation UpdateUser($id: ID!, $name: String, $email: String) {
    updateUser(id: $id, name: $name, email: $email) {
      id
      name
      email
    }
  }
`;

const DELETE_USER = gql`
  mutation DeleteUser($id: ID!) {
    deleteUser(id: $id)
  }
`;

// React component using GraphQL
import React from 'react';

const UserList = () => {
  const { loading, error, data, refetch } = useQuery(GET_USERS, {
    pollInterval: 30000, // Poll every 30 seconds
    notifyOnNetworkStatusChange: true,
  });

  const [createUser] = useMutation(CREATE_USER, {
    update(cache, { data: { createUser } }) {
      const existingUsers = cache.readQuery({ query: GET_USERS });
      cache.writeQuery({
        query: GET_USERS,
        data: {
          users: [...existingUsers.users, createUser],
        },
      });
    },
  });

  const [deleteUser] = useMutation(DELETE_USER, {
    update(cache, { data: { deleteUser } }, { variables }) {
      if (deleteUser) {
        cache.evict({ id: `User:${variables.id}` });
        cache.gc();
      }
    },
  });

  const handleCreateUser = async (userData) => {
    try {
      await createUser({
        variables: userData,
      });
    } catch (error) {
      console.error('Error creating user:', error);
    }
  };

  const handleDeleteUser = async (id) => {
    try {
      await deleteUser({
        variables: { id },
      });
    } catch (error) {
      console.error('Error deleting user:', error);
    }
  };

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <h2>Users</h2>
      <button onClick={() => refetch()}>Refresh</button>
      <ul>
        {data.users.map(user => (
          <li key={user.id}>
            <div>
              <strong>{user.name}</strong> - {user.email}
              <button onClick={() => handleDeleteUser(user.id)}>
                Delete
              </button>
            </div>
            <div>Posts: {user.posts.length}</div>
          </li>
        ))}
      </ul>
    </div>
  );
};

export default UserList;
```

### GraphQL with Custom Hooks

```javascript
// Custom GraphQL hooks
import { useQuery, useMutation } from '@apollo/client';

export const useUsers = (options = {}) => {
  return useQuery(GET_USERS, {
    errorPolicy: 'all',
    ...options,
  });
};

export const useUser = (id, options = {}) => {
  return useQuery(GET_USER, {
    variables: { id },
    skip: !id,
    errorPolicy: 'all',
    ...options,
  });
};

export const useCreateUser = (options = {}) => {
  return useMutation(CREATE_USER, {
    update(cache, { data: { createUser } }) {
      try {
        const existingUsers = cache.readQuery({ query: GET_USERS });
        cache.writeQuery({
          query: GET_USERS,
          data: {
            users: [...existingUsers.users, createUser],
          },
        });
      } catch (error) {
        // Query might not be in cache yet
        console.warn('Could not update cache:', error);
      }
    },
    ...options,
  });
};

export const useUpdateUser = (options = {}) => {
  return useMutation(UPDATE_USER, {
    ...options,
  });
};

export const useDeleteUser = (options = {}) => {
  return useMutation(DELETE_USER, {
    update(cache, { data: { deleteUser } }, { variables }) {
      if (deleteUser) {
        cache.evict({ id: `User:${variables.id}` });
        cache.gc();
      }
    },
    ...options,
  });
};

// Usage in component
const UserManagement = () => {
  const { data: users, loading, error, refetch } = useUsers();
  const [createUser, { loading: creating }] = useCreateUser();
  const [updateUser, { loading: updating }] = useUpdateUser();
  const [deleteUser, { loading: deleting }] = useDeleteUser();

  // Component logic here...
};
```

## Performance Comparison

### Network Efficiency

**REST:**
```javascript
// Multiple requests needed for related data
const fetchUserWithPosts = async (userId) => {
  const user = await fetch(`/api/users/${userId}`);
  const posts = await fetch(`/api/users/${userId}/posts`);
  const comments = await fetch(`/api/posts/${posts[0].id}/comments`);
  
  // 3 separate network requests
  return { user, posts, comments };
};
```

**GraphQL:**
```javascript
// Single request for all related data
const GET_USER_WITH_POSTS_AND_COMMENTS = gql`
  query GetUserWithPostsAndComments($userId: ID!) {
    user(id: $userId) {
      id
      name
      email
      posts {
        id
        title
        content
        comments {
          id
          content
          author {
            name
          }
        }
      }
    }
  }
`;

// Only 1 network request
const { data } = useQuery(GET_USER_WITH_POSTS_AND_COMMENTS, {
  variables: { userId }
});
```

### Caching Strategies

**REST Caching:**
```javascript
// HTTP caching with service worker
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/users')) {
    event.respondWith(
      caches.open('api-cache').then((cache) => {
        return cache.match(event.request).then((response) => {
          if (response) {
            // Serve from cache
            fetch(event.request).then((fetchResponse) => {
              cache.put(event.request, fetchResponse.clone());
            });
            return response;
          }
          // Fetch and cache
          return fetch(event.request).then((fetchResponse) => {
            cache.put(event.request, fetchResponse.clone());
            return fetchResponse;
          });
        });
      })
    );
  }
});
```

**GraphQL Caching:**
```javascript
// Apollo Client normalized cache
const cache = new InMemoryCache({
  typePolicies: {
    User: {
      fields: {
        posts: {
          merge(existing = [], incoming) {
            return [...existing, ...incoming];
          },
        },
      },
    },
    Post: {
      fields: {
        comments: {
          merge(existing = [], incoming) {
            return [...existing, ...incoming];
          },
        },
      },
    },
  },
});

// Automatic cache updates
const [updateUser] = useMutation(UPDATE_USER, {
  update(cache, { data: { updateUser } }) {
    cache.modify({
      id: cache.identify(updateUser),
      fields: {
        name() {
          return updateUser.name;
        },
        email() {
          return updateUser.email;
        },
      },
    });
  },
});
```

## Use Cases

### When to Choose REST

1. **Simple CRUD Operations**: When your application primarily performs basic create, read, update, delete operations
2. **Caching Requirements**: When HTTP caching is crucial for performance
3. **File Uploads**: When you need to handle file uploads frequently
4. **Team Familiarity**: When your team is more familiar with REST
5. **Third-party Integrations**: When integrating with existing REST APIs
6. **Microservices**: When working with microservices architecture where each service has its own API

**Example Scenario:**
```javascript
// E-commerce product catalog
// REST is ideal for simple operations
GET /api/products?category=electronics&page=1
GET /api/products/123
POST /api/products
PUT /api/products/123
DELETE /api/products/123
```

### When to Choose GraphQL

1. **Complex Data Requirements**: When clients need different subsets of data
2. **Mobile Applications**: When bandwidth is limited and over-fetching is costly
3. **Rapid Frontend Development**: When frontend teams need flexibility
4. **Real-time Features**: When you need subscriptions for live updates
5. **Multiple Clients**: When serving web, mobile, and other clients with different needs
6. **Evolving APIs**: When your API schema changes frequently

**Example Scenario:**
```javascript
// Social media dashboard
// GraphQL allows flexible data fetching
query DashboardData($userId: ID!) {
  user(id: $userId) {
    name
    avatar
    posts(limit: 10) {
      id
      content
      likes
      comments(limit: 3) {
        content
        author {
          name
        }
      }
    }
    notifications(unreadOnly: true) {
      id
      message
      createdAt
    }
  }
}
```

## Pros and Cons

### REST Advantages

1. **Simplicity**: Easy to understand and implement
2. **HTTP Caching**: Leverages existing HTTP caching mechanisms
3. **Stateless**: Each request is independent
4. **Mature Ecosystem**: Extensive tooling and libraries
5. **Debugging**: Easy to debug with standard HTTP tools
6. **File Handling**: Native support for file uploads

### REST Disadvantages

1. **Over-fetching**: Often returns more data than needed
2. **Under-fetching**: May require multiple requests for related data
3. **Versioning**: API versioning can become complex
4. **Rigid Structure**: Fixed endpoints limit flexibility

### GraphQL Advantages

1. **Flexible Queries**: Clients request exactly what they need
2. **Single Endpoint**: Simplifies API management
3. **Strong Typing**: Schema provides clear contract
4. **Real-time**: Built-in subscription support
5. **Introspection**: Self-documenting APIs
6. **Backward Compatibility**: Easy to evolve without versioning

### GraphQL Disadvantages

1. **Complexity**: Higher learning curve
2. **Caching**: More complex than HTTP caching
3. **Query Complexity**: Risk of expensive queries
4. **File Uploads**: Requires additional specifications
5. **Overhead**: Can be overkill for simple applications

## Migration Strategies

### REST to GraphQL Migration

```javascript
// Gradual migration approach
// 1. Start with GraphQL wrapper around REST endpoints

const resolvers = {
  Query: {
    users: async () => {
      const response = await fetch('/api/users');
      return response.json();
    },
    user: async (_, { id }) => {
      const response = await fetch(`/api/users/${id}`);
      return response.json();
    },
  },
  User: {
    posts: async (user) => {
      const response = await fetch(`/api/users/${user.id}/posts`);
      return response.json();
    },
  },
};

// 2. Gradually replace REST calls with native GraphQL resolvers
const resolvers = {
  Query: {
    users: async (_, __, { dataSources }) => {
      return dataSources.userAPI.getAllUsers();
    },
  },
};

// 3. Frontend migration with feature flags
const useGraphQL = process.env.REACT_APP_USE_GRAPHQL === 'true';

const UserList = () => {
  if (useGraphQL) {
    return <GraphQLUserList />;
  }
  return <RESTUserList />;
};
```

### GraphQL to REST Migration

```javascript
// Extract REST endpoints from GraphQL schema
// 1. Identify common query patterns
const commonQueries = [
  'users',
  'user(id: ID!)',
  'posts',
  'post(id: ID!)',
];

// 2. Create REST endpoints for common patterns
app.get('/api/users', (req, res) => {
  // Implementation that matches GraphQL users query
});

app.get('/api/users/:id', (req, res) => {
  // Implementation that matches GraphQL user(id) query
});

// 3. Gradual frontend migration
const UserComponent = () => {
  const useREST = process.env.REACT_APP_USE_REST === 'true';
  
  if (useREST) {
    const { data, loading, error } = useFetch('/api/users');
    // REST implementation
  } else {
    const { data, loading, error } = useQuery(GET_USERS);
    // GraphQL implementation
  }
};
```

## Common Beginner Doubts

### Q1: "Should I always choose GraphQL over REST for new projects?"

**Answer**: Not necessarily. The choice depends on your specific requirements:

- **Choose GraphQL** if you have complex data relationships, multiple client types, or need real-time features
- **Choose REST** if you have simple CRUD operations, need extensive caching, or your team is more familiar with REST

### Q2: "Is GraphQL faster than REST?"

**Answer**: It depends on the use case:

- **GraphQL can be faster** when you need to fetch related data (eliminates multiple round trips)
- **REST can be faster** for simple operations and benefits from HTTP caching
- **Performance depends more on implementation** than the technology choice

### Q3: "Can I use both REST and GraphQL in the same application?"

**Answer**: Yes, absolutely! Many applications use a hybrid approach:

```javascript
// Use REST for simple operations
const uploadFile = async (file) => {
  const formData = new FormData();
  formData.append('file', file);
  return fetch('/api/upload', {
    method: 'POST',
    body: formData,
  });
};

// Use GraphQL for complex data fetching
const { data } = useQuery(GET_USER_DASHBOARD, {
  variables: { userId }
});
```

### Q4: "How do I handle authentication with GraphQL?"

**Answer**: Authentication works similarly to REST:

```javascript
// JWT token in headers
const authLink = setContext((_, { headers }) => {
  const token = localStorage.getItem('token');
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : "",
    }
  };
});

// Or in GraphQL context
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => {
    const token = req.headers.authorization || '';
    const user = getUser(token);
    return { user };
  },
});
```

### Q5: "How do I handle errors in GraphQL vs REST?"

**Answer**: Error handling differs between the two:

**REST Error Handling:**
```javascript
try {
  const response = await fetch('/api/users');
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  const data = await response.json();
} catch (error) {
  console.error('Request failed:', error);
}
```

**GraphQL Error Handling:**
```javascript
const { data, error } = useQuery(GET_USERS);

if (error) {
  // GraphQL errors array
  error.graphQLErrors.forEach(({ message, locations, path }) => {
    console.error(`GraphQL error: ${message}`);
  });
  
  // Network errors
  if (error.networkError) {
    console.error('Network error:', error.networkError);
  }
}
```

### Q6: "How do I implement pagination with GraphQL?"

**Answer**: GraphQL supports multiple pagination patterns:

```javascript
// Cursor-based pagination (recommended)
const GET_POSTS = gql`
  query GetPosts($first: Int!, $after: String) {
    posts(first: $first, after: $after) {
      edges {
        node {
          id
          title
          content
        }
        cursor
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
`;

// Offset-based pagination
const GET_POSTS_OFFSET = gql`
  query GetPosts($limit: Int!, $offset: Int!) {
    posts(limit: $limit, offset: $offset) {
      id
      title
      content
    }
    postsCount
  }
`;
```

## Best Practices

### REST Best Practices

1. **Use Proper HTTP Methods**
```javascript
// Correct HTTP method usage
GET    /api/users           // Retrieve
POST   /api/users           // Create
PUT    /api/users/123       // Update (full)
PATCH  /api/users/123       // Update (partial)
DELETE /api/users/123       // Delete
```

2. **Implement Proper Error Handling**
```javascript
const apiCall = async (url, options = {}) => {
  try {
    const response = await fetch(url, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(errorData.message || `HTTP ${response.status}`);
    }

    return await response.json();
  } catch (error) {
    console.error('API call failed:', error);
    throw error;
  }
};
```

3. **Use Consistent Response Formats**
```javascript
// Consistent API response structure
{
  "success": true,
  "data": {
    "users": [...]
  },
  "meta": {
    "total": 100,
    "page": 1,
    "limit": 10
  }
}

// Error response structure
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": {
      "field": "email"
    }
  }
}
```

### GraphQL Best Practices

1. **Design Efficient Queries**
```javascript
// Good: Request only needed fields
const GET_USER_SUMMARY = gql`
  query GetUserSummary($id: ID!) {
    user(id: $id) {
      id
      name
      email
      postsCount
    }
  }
`;

// Avoid: Over-fetching unnecessary data
const GET_USER_EVERYTHING = gql`
  query GetUserEverything($id: ID!) {
    user(id: $id) {
      id
      name
      email
      bio
      avatar
      createdAt
      updatedAt
      posts {
        id
        title
        content
        createdAt
        comments {
          id
          content
          author {
            id
            name
            email
          }
        }
      }
    }
  }
`;
```

2. **Implement Query Complexity Analysis**
```javascript
// Prevent expensive queries
import { createComplexityLimitRule } from 'graphql-query-complexity';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    createComplexityLimitRule(1000, {
      maximumComplexity: 1000,
      variables: {},
      createError: (max, actual) => {
        return new Error(`Query complexity ${actual} exceeds maximum ${max}`);
      },
    }),
  ],
});
```

3. **Use DataLoader for N+1 Problem**
```javascript
import DataLoader from 'dataloader';

// Batch loading to prevent N+1 queries
const userLoader = new DataLoader(async (userIds) => {
  const users = await User.findByIds(userIds);
  return userIds.map(id => users.find(user => user.id === id));
});

const resolvers = {
  Post: {
    author: (post) => userLoader.load(post.authorId),
  },
};
```

4. **Implement Proper Error Handling**
```javascript
import { AuthenticationError, ForbiddenError, UserInputError } from 'apollo-server-express';

const resolvers = {
  Mutation: {
    createPost: async (_, { input }, { user }) => {
      if (!user) {
        throw new AuthenticationError('You must be logged in');
      }
      
      if (!input.title || input.title.trim().length === 0) {
        throw new UserInputError('Title is required', {
          invalidArgs: ['title'],
        });
      }
      
      try {
        return await Post.create({ ...input, authorId: user.id });
      } catch (error) {
        throw new Error('Failed to create post');
      }
    },
  },
};
```

## Conclusion

Both REST and GraphQL are powerful approaches for API design, each with distinct advantages and trade-offs. The choice between them should be based on your specific project requirements, team expertise, and long-term goals.

**Choose REST when:**
- Building simple CRUD applications
- HTTP caching is critical
- Working with file uploads
- Team has strong REST experience
- Integrating with existing REST services

**Choose GraphQL when:**
- Dealing with complex, interconnected data
- Supporting multiple client types with different data needs
- Building real-time applications
- Rapid frontend development is a priority
- API evolution and backward compatibility are important

**Key Takeaways:**

1. **Not Mutually Exclusive**: You can use both REST and GraphQL in the same application
2. **Performance Depends on Implementation**: Both can be optimized for excellent performance
3. **Team and Project Context Matters**: Consider your team's expertise and project requirements
4. **Evolution Path**: You can migrate between approaches as your needs change
5. **Tooling and Ecosystem**: Both have mature tooling, though REST has a longer history

Ultimately, the best choice is the one that aligns with your project's specific needs, team capabilities, and long-term architectural goals. Understanding both approaches deeply will help you make informed decisions and potentially leverage the strengths of each in different parts of your application.
