# GitHub Discussions GraphQL API Guide

## Overview
This guide covers how to fetch GitHub discussions using the GraphQL API, with a focus on filtering by category and labels.

## 1. Basic Discussion Query Structure

### Available Fields for Discussions
```graphql
# Common fields available for discussions
discussion {
  id                    # GraphQL node ID
  number               # Discussion number (e.g., #123)
  title                # Discussion title
  body                 # Main content/body text
  bodyHTML             # HTML version of body
  bodyText             # Plain text version of body
  url                  # Full URL to the discussion
  createdAt            # Creation timestamp
  updatedAt            # Last update timestamp
  closed               # Boolean - is discussion closed
  closedAt             # When discussion was closed (if applicable)
  locked               # Boolean - is discussion locked
  
  # Author information
  author {
    login              # Username
    avatarUrl
    url
  }
  
  # Category information
  category {
    id
    name
    slug
    emoji
    emojiHTML
  }
  
  # Labels
  labels(first: 100) {
    nodes {
      id
      name
      color
      description
    }
  }
  
  # Comments/replies
  comments(first: 10) {
    totalCount
    nodes {
      id
      body
      createdAt
      author {
        login
      }
    }
  }
  
  # Reactions
  reactions(first: 10) {
    totalCount
    nodes {
      content           # THUMBS_UP, THUMBS_DOWN, LAUGH, etc.
      user {
        login
      }
    }
  }
}
```

## 2. Fetching Discussions by Category

### Query to get discussions from a specific category
```graphql
query GetDiscussionsByCategory($owner: String!, $repo: String!, $categoryId: ID!) {
  repository(owner: $owner, name: $repo) {
    discussions(first: 25, categoryId: $categoryId) {
      totalCount
      pageInfo {
        hasNextPage
        endCursor
      }
      nodes {
        id
        number
        title
        url
        createdAt
        author {
          login
        }
        category {
          name
        }
        labels(first: 10) {
          nodes {
            name
            color
          }
        }
      }
    }
  }
}
```

### First, get the category ID
```graphql
query GetCategories($owner: String!, $repo: String!) {
  repository(owner: $owner, name: $repo) {
    discussionCategories(first: 100) {
      nodes {
        id
        name
        slug
        description
        emoji
      }
    }
  }
}
```

## 3. Filtering Discussions by Labels

### Query discussions with specific labels
```graphql
query GetDiscussionsByLabel($owner: String!, $repo: String!, $labels: [String!]) {
  repository(owner: $owner, name: $repo) {
    discussions(first: 25, labels: $labels) {
      totalCount
      pageInfo {
        hasNextPage
        endCursor
      }
      nodes {
        id
        number
        title
        body
        url
        createdAt
        category {
          name
        }
        labels(first: 20) {
          nodes {
            name
            color
            description
          }
        }
      }
    }
  }
}
```

## 4. Combined Filtering (Category + Labels)

### Query discussions with both category and label filters
```graphql
query GetFeatureRequests($owner: String!, $repo: String!, $categoryId: ID!, $labels: [String!]) {
  repository(owner: $owner, name: $repo) {
    discussions(
      first: 50,
      categoryId: $categoryId,
      labels: $labels,
      orderBy: {field: CREATED_AT, direction: DESC}
    ) {
      totalCount
      pageInfo {
        hasNextPage
        endCursor
      }
      nodes {
        id
        number
        title
        body
        url
        createdAt
        updatedAt
        author {
          login
          avatarUrl
        }
        category {
          id
          name
          slug
        }
        labels(first: 20) {
          totalCount
          nodes {
            id
            name
            color
            description
          }
        }
        comments {
          totalCount
        }
        reactions {
          totalCount
        }
      }
    }
  }
}
```

## 5. Practical Examples for Feature Requests

### Example 1: Get all feature requests with a specific integration label
```graphql
query GetIntegrationFeatureRequests {
  repository(owner: "home-assistant", name: "feature-requests") {
    discussions(
      first: 30,
      labels: ["integration: hue"],
      orderBy: {field: UPDATED_AT, direction: DESC}
    ) {
      totalCount
      nodes {
        id
        number
        title
        url
        createdAt
        labels(first: 10) {
          nodes {
            name
          }
        }
      }
    }
  }
}
```

### Example 2: Get feature requests from multiple integrations
```graphql
query GetMultipleIntegrationRequests {
  repository(owner: "home-assistant", name: "feature-requests") {
    discussions(
      first: 50,
      labels: ["integration: hue", "integration: mqtt", "integration: zigbee"],
      orderBy: {field: CREATED_AT, direction: DESC}
    ) {
      totalCount
      nodes {
        id
        number
        title
        url
        labels(first: 20) {
          nodes {
            name
            color
          }
        }
      }
    }
  }
}
```

### Example 3: Search discussions with text query + labels
```graphql
query SearchFeatureRequests($searchQuery: String!) {
  search(
    query: $searchQuery,
    type: DISCUSSION,
    first: 25
  ) {
    discussionCount
    pageInfo {
      hasNextPage
      endCursor
    }
    nodes {
      ... on Discussion {
        id
        number
        title
        body
        url
        repository {
          nameWithOwner
        }
        category {
          name
        }
        labels(first: 10) {
          nodes {
            name
            color
          }
        }
      }
    }
  }
}

# Variables example:
{
  "searchQuery": "repo:home-assistant/feature-requests label:\"integration: hue\" light"
}
```

### Example 4: Get discussions with pagination
```graphql
query GetPaginatedDiscussions($owner: String!, $repo: String!, $cursor: String) {
  repository(owner: $owner, name: $repo) {
    discussions(
      first: 25,
      after: $cursor,
      labels: ["integration: mqtt"],
      orderBy: {field: CREATED_AT, direction: DESC}
    ) {
      totalCount
      pageInfo {
        hasNextPage
        hasPreviousPage
        startCursor
        endCursor
      }
      nodes {
        id
        number
        title
        url
        createdAt
        labels(first: 10) {
          nodes {
            name
          }
        }
      }
    }
  }
}
```

## 6. Getting Discussion IDs and URLs

### Different types of IDs available:
1. **GraphQL Node ID** (`id`): Base64 encoded ID used in GraphQL mutations
2. **Discussion Number** (`number`): Integer number shown in UI (e.g., #123)
3. **URL** (`url`): Full GitHub URL to the discussion

### Example: Get all ID types
```graphql
query GetDiscussionIdentifiers {
  repository(owner: "home-assistant", name: "feature-requests") {
    discussion(number: 123) {
      id           # e.g., "D_kwDOBq2Y884APqYs"
      number       # e.g., 123
      url          # e.g., "https://github.com/home-assistant/feature-requests/discussions/123"
      resourcePath # e.g., "/home-assistant/feature-requests/discussions/123"
    }
  }
}
```

## 7. Advanced Filtering Examples

### Example: Feature requests with multiple criteria
```graphql
query ComplexFeatureRequestQuery {
  repository(owner: "home-assistant", name: "feature-requests") {
    # Get specific category first
    category: discussionCategory(slug: "feature-requests") {
      id
      name
    }
    
    # Then get discussions
    discussions(
      first: 50,
      categoryId: "DIC_kwDOBq2Y884B-Jmh",  # Use actual category ID
      labels: ["integration: hue", "enhancement"],
      orderBy: {field: UPDATED_AT, direction: DESC}
    ) {
      totalCount
      nodes {
        id
        number
        title
        body
        url
        createdAt
        updatedAt
        
        # Check if discussion is answered
        isAnswered
        
        # Get the answer if exists
        answer {
          id
          body
          createdAt
          author {
            login
          }
        }
        
        # All labels
        labels(first: 20) {
          nodes {
            name
            color
            description
          }
        }
        
        # Participants
        participants(first: 10) {
          nodes {
            login
            avatarUrl
          }
        }
      }
    }
  }
}
```

## 8. Using Variables in Practice

### Python example using requests:
```python
import requests
import json

# GitHub token with read:discussion scope
token = "YOUR_GITHUB_TOKEN"

# GraphQL endpoint
url = "https://api.github.com/graphql"

# Headers
headers = {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/json"
}

# Query
query = """
query GetIntegrationDiscussions($owner: String!, $repo: String!, $labels: [String!]!) {
  repository(owner: $owner, name: $repo) {
    discussions(first: 25, labels: $labels) {
      totalCount
      nodes {
        id
        number
        title
        url
        labels(first: 10) {
          nodes {
            name
          }
        }
      }
    }
  }
}
"""

# Variables
variables = {
    "owner": "home-assistant",
    "repo": "feature-requests",
    "labels": ["integration: hue", "integration: mqtt"]
}

# Make request
response = requests.post(
    url,
    headers=headers,
    json={"query": query, "variables": variables}
)

data = response.json()
print(json.dumps(data, indent=2))
```

## Notes and Tips

1. **Rate Limiting**: GitHub GraphQL API has rate limits. Use pagination and request only needed fields.

2. **Label Filtering**: When using `labels` parameter, it acts as an OR operation (discussions with any of the specified labels).

3. **Category vs Labels**: Categories are mutually exclusive (a discussion belongs to one category), while multiple labels can be applied.

4. **Search vs Repository Query**: Use `search` for text-based queries across repositories, use `repository.discussions` for structured queries within a specific repo.

5. **Performance**: Request only the fields you need. Avoid deeply nested queries with large limits.

6. **Authentication**: Some fields may require authentication. Use a personal access token with appropriate scopes.