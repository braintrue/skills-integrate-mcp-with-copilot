# Implementation Proposal: GitHub Discussions Support for MCP Server

## Issue Reference
**GitHub Issue**: #1171 - Tool for Creating and Modifying GitHub Discussions via MCP Server  
**Repository**: github/github-mcp-server  
**Priority**: High  
**Type**: Feature Enhancement  

## Executive Summary

This proposal outlines the implementation of GitHub Discussions support in the GitHub MCP Server. This feature will enable AI assistants and automated workflows to create, read, update, and manage GitHub Discussions programmatically through the MCP protocol.

## Motivation

### Business Value
- **Community Automation**: Enable automated community engagement at scale
- **Knowledge Management**: Facilitate AI-driven knowledge base creation
- **Workflow Integration**: Connect Discussions with existing development workflows
- **Competitive Parity**: Match feature completeness of other GitHub API tools

### User Needs
From issue analysis and community feedback:
1. Post automated meeting summaries and updates
2. Create FAQ discussions from common issues
3. Moderate community discussions efficiently
4. Generate AI-assisted responses to questions
5. Cross-link issues and discussions automatically

### Current Gap
The MCP server supports Issues, PRs, and Projects but not Discussions, despite Discussions being a core GitHub feature used by thousands of repositories for community engagement.

## Technical Design

### Architecture Overview

```
┌─────────────────┐
│   MCP Client    │
│  (AI Assistant) │
└────────┬────────┘
         │
         │ MCP Protocol
         ▼
┌─────────────────┐
│  MCP Server     │
│  (GitHub)       │
├─────────────────┤
│  Discussions    │
│  Handler        │
└────────┬────────┘
         │
         │ GraphQL API
         ▼
┌─────────────────┐
│  GitHub API     │
│  Discussions    │
└─────────────────┘
```

### API Selection: GraphQL vs REST

**Decision**: Use GraphQL API

**Rationale**:
1. GitHub Discussions is primarily exposed via GraphQL
2. Richer query capabilities (nested replies, pagination)
3. Flexible field selection reduces bandwidth
4. Better type safety and documentation
5. Existing GitHub MCP server uses GraphQL for Projects

### Tools to Implement

#### Phase 1: Read Operations

##### 1. list_discussions
**Description**: List discussions in a repository

**Parameters**:
```typescript
{
  owner: string;              // Required: Repository owner
  repo: string;               // Required: Repository name
  category_id?: string;       // Optional: Filter by category
  answered?: boolean;         // Optional: Filter by answered status
  first?: number;             // Optional: Number of results (default 30, max 100)
  after?: string;             // Optional: Pagination cursor
  orderBy?: {
    field: 'CREATED_AT' | 'UPDATED_AT';
    direction: 'ASC' | 'DESC';
  };
}
```

**Response**:
```typescript
{
  discussions: [
    {
      number: number;
      title: string;
      body: string;
      author: string;
      category: string;
      answered: boolean;
      answerChosenAt?: string;
      createdAt: string;
      updatedAt: string;
      url: string;
      upvoteCount: number;
      commentCount: number;
    }
  ],
  pageInfo: {
    hasNextPage: boolean;
    endCursor: string;
  }
}
```

**GraphQL Query**:
```graphql
query ListDiscussions($owner: String!, $repo: String!, $first: Int, $after: String) {
  repository(owner: $owner, name: $repo) {
    discussions(first: $first, after: $after) {
      nodes {
        number
        title
        body
        author { login }
        category { name }
        answered
        answerChosenAt
        createdAt
        updatedAt
        url
        upvoteCount
        comments { totalCount }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

##### 2. get_discussion
**Description**: Get a specific discussion with all details

**Parameters**:
```typescript
{
  owner: string;              // Required: Repository owner
  repo: string;               // Required: Repository name
  discussion_number: number;  // Required: Discussion number
}
```

**Response**:
```typescript
{
  number: number;
  title: string;
  body: string;
  bodyHTML: string;
  author: string;
  category: string;
  answered: boolean;
  answer?: {
    body: string;
    author: string;
    createdAt: string;
  };
  createdAt: string;
  updatedAt: string;
  url: string;
  upvoteCount: number;
  comments: [
    {
      id: string;
      body: string;
      author: string;
      createdAt: string;
      upvoteCount: number;
      replies: [...]  // Nested replies
    }
  ]
}
```

##### 3. get_discussion_categories
**Description**: List available discussion categories for a repository

**Parameters**:
```typescript
{
  owner: string;  // Required: Repository owner
  repo: string;   // Required: Repository name
}
```

**Response**:
```typescript
{
  categories: [
    {
      id: string;
      name: string;
      emoji: string;
      description: string;
      isAnswerable: boolean;
    }
  ]
}
```

#### Phase 2: Write Operations

##### 4. create_discussion
**Description**: Create a new discussion

**Parameters**:
```typescript
{
  owner: string;        // Required: Repository owner
  repo: string;         // Required: Repository name
  category_id: string;  // Required: Discussion category ID
  title: string;        // Required: Discussion title (max 256 chars)
  body: string;         // Required: Discussion body (markdown)
}
```

**Response**:
```typescript
{
  discussion: {
    number: number;
    title: string;
    url: string;
    id: string;
  }
}
```

**GraphQL Mutation**:
```graphql
mutation CreateDiscussion($repositoryId: ID!, $categoryId: ID!, $title: String!, $body: String!) {
  createDiscussion(input: {
    repositoryId: $repositoryId,
    categoryId: $categoryId,
    title: $title,
    body: $body
  }) {
    discussion {
      number
      title
      url
      id
    }
  }
}
```

**Validation**:
- Title: 1-256 characters
- Body: Required, max 65536 characters
- Category ID: Must be valid for the repository
- Repository must have Discussions enabled

##### 5. update_discussion
**Description**: Update an existing discussion

**Parameters**:
```typescript
{
  owner: string;             // Required: Repository owner
  repo: string;              // Required: Repository name
  discussion_number: number; // Required: Discussion number
  title?: string;            // Optional: New title
  body?: string;             // Optional: New body
  category_id?: string;      // Optional: New category ID
}
```

**Response**:
```typescript
{
  discussion: {
    number: number;
    title: string;
    url: string;
  }
}
```

**Permissions Required**: 
- Author of the discussion OR
- Repository write access

##### 6. add_discussion_comment
**Description**: Add a comment to a discussion

**Parameters**:
```typescript
{
  owner: string;             // Required: Repository owner
  repo: string;              // Required: Repository name
  discussion_number: number; // Required: Discussion number
  body: string;              // Required: Comment body (markdown)
  reply_to_id?: string;      // Optional: Comment ID to reply to
}
```

**Response**:
```typescript
{
  comment: {
    id: string;
    url: string;
    createdAt: string;
  }
}
```

#### Phase 3: Advanced Operations

##### 7. mark_discussion_answer
**Description**: Mark a comment as the answer to a discussion

**Parameters**:
```typescript
{
  owner: string;
  repo: string;
  discussion_number: number;
  comment_id: string;
}
```

**Use Case**: Automated Q&A resolution

##### 8. lock_discussion
**Description**: Lock a discussion to prevent new comments

**Parameters**:
```typescript
{
  owner: string;
  repo: string;
  discussion_number: number;
  lock_reason?: 'OFF_TOPIC' | 'TOO_HEATED' | 'RESOLVED' | 'SPAM';
}
```

**Use Case**: Moderation workflows

##### 9. toggle_discussion_upvote
**Description**: Upvote or remove upvote from a discussion

**Parameters**:
```typescript
{
  owner: string;
  repo: string;
  discussion_number: number;
  upvote: boolean;  // true to upvote, false to remove upvote
}
```

## Implementation Plan

### Phase 1: Foundation (Week 1)

**Goals**:
- Set up base structure
- Implement read-only operations
- Create comprehensive tests

**Tasks**:
1. Create `discussions.go` handler file
2. Define GraphQL query templates
3. Implement `list_discussions`
4. Implement `get_discussion`
5. Implement `get_discussion_categories`
6. Write unit tests (80%+ coverage)
7. Write integration tests with mock API
8. Add tool documentation

**Deliverables**:
- Working read-only tools
- Test suite
- Tool documentation

### Phase 2: Write Operations (Week 2)

**Goals**:
- Enable discussion creation and updates
- Add comment functionality
- Ensure proper error handling

**Tasks**:
1. Implement `create_discussion`
2. Add input validation layer
3. Implement `update_discussion`
4. Implement `add_discussion_comment`
5. Add permission checking
6. Write integration tests
7. Add error handling tests
8. Update documentation

**Deliverables**:
- Full CRUD operations
- Enhanced test coverage
- Complete documentation

### Phase 3: Advanced Features (Week 3)

**Goals**:
- Add moderation capabilities
- Implement answer marking
- Polish and optimize

**Tasks**:
1. Implement `mark_discussion_answer`
2. Implement `lock_discussion`
3. Implement `toggle_discussion_upvote`
4. Add rate limit handling
5. Optimize GraphQL queries
6. Performance testing
7. Security audit
8. Final documentation pass

**Deliverables**:
- Complete feature set
- Performance optimizations
- Security validation

### Phase 4: Documentation & Release (Week 4)

**Goals**:
- Complete documentation
- Create examples
- Prepare for release

**Tasks**:
1. Write comprehensive README section
2. Create example workflows
3. Add troubleshooting guide
4. Create demo video/GIF
5. Update main documentation
6. Prepare release notes
7. Community announcement

**Deliverables**:
- Complete documentation
- Example workflows
- Release announcement

## Testing Strategy

### Unit Tests

```go
// Test successful discussion creation
func TestCreateDiscussion_Success(t *testing.T) {
  // Setup mock GraphQL client
  // Call createDiscussion
  // Assert response
}

// Test validation errors
func TestCreateDiscussion_InvalidTitle(t *testing.T) {
  // Test title too long
  // Test empty title
  // Assert appropriate errors
}

// Test permission errors
func TestCreateDiscussion_NoPermission(t *testing.T) {
  // Mock permission denied response
  // Assert proper error handling
}
```

### Integration Tests

```go
// Test with actual GitHub repository
func TestDiscussionsIntegration(t *testing.T) {
  if testing.Short() {
    t.Skip("Skipping integration test")
  }
  
  // Create test discussion
  // Verify it exists
  // Update discussion
  // Add comment
  // Clean up
}
```

### Example Prompts for Testing

1. "List all open questions in the Q&A category"
2. "Create a discussion titled 'Feature Request: Dark Mode' in the Ideas category"
3. "Show me the most upvoted discussions from this month"
4. "Reply to discussion #42 with implementation suggestions"
5. "Mark comment X as the answer to discussion #15"

## Security Considerations

### Input Validation
- Sanitize markdown input to prevent XSS
- Validate discussion numbers and IDs
- Check title and body length limits
- Verify category IDs exist

### Authentication & Authorization
- Validate GitHub token permissions
- Check repository access before operations
- Verify user can modify discussions they don't own
- Respect repository Discussions settings

### Rate Limiting
- Implement retry logic with exponential backoff
- Cache category listings to reduce API calls
- Respect GitHub API rate limits
- Add request throttling for bulk operations

### Error Handling
```go
// Example error handling pattern
func (h *DiscussionsHandler) CreateDiscussion(params CreateDiscussionParams) (*Discussion, error) {
  // Validate input
  if err := validateDiscussionInput(params); err != nil {
    return nil, fmt.Errorf("validation error: %w", err)
  }
  
  // Check permissions
  if !hasDiscussionAccess(params.Owner, params.Repo) {
    return nil, ErrNoPermission
  }
  
  // Execute mutation with retry
  result, err := h.executeWithRetry(createDiscussionMutation, params)
  if err != nil {
    return nil, fmt.Errorf("failed to create discussion: %w", err)
  }
  
  return result, nil
}
```

## Documentation Requirements

### Tool Documentation

Each tool needs:
1. **Description**: What the tool does
2. **Parameters**: Required and optional parameters with types
3. **Returns**: Response structure and data types
4. **Examples**: 3-5 example prompts
5. **Errors**: Common error cases and solutions
6. **Permissions**: Required GitHub permissions

### Example Workflows

Document these workflows:
1. **Community Q&A**: Automated question responses
2. **Release Announcements**: Create discussions for new releases
3. **Meeting Minutes**: Post meeting summaries
4. **Feature Voting**: Track feature requests via upvotes
5. **Knowledge Base**: Convert issues to discussion documentation

### Troubleshooting Guide

Common issues:
- "Discussions not enabled for repository"
- "Invalid category ID"
- "Permission denied"
- "Rate limit exceeded"
- "Discussion not found"

## Success Metrics

### Functional Metrics
- ✅ All tools work correctly
- ✅ 90%+ test coverage
- ✅ Zero critical bugs in production
- ✅ Response time < 2 seconds for typical operations

### Adoption Metrics
- Track tool usage via telemetry
- Monitor error rates
- Gather user feedback
- Measure API success rate

### Community Metrics
- GitHub issue reactions/comments
- Pull request engagement
- Documentation views
- Community contributions

## Risks & Mitigation

### Technical Risks

**Risk**: GraphQL API changes breaking implementation  
**Mitigation**: Use versioned API, implement robust error handling, add monitoring

**Risk**: Rate limiting affecting user experience  
**Mitigation**: Implement caching, request batching, clear rate limit messaging

**Risk**: Complex nested query performance  
**Mitigation**: Optimize queries, implement pagination, add query depth limits

### Product Risks

**Risk**: Low adoption due to limited use cases  
**Mitigation**: Create compelling examples, gather early feedback, iterate on workflows

**Risk**: Security vulnerabilities in user input handling  
**Mitigation**: Comprehensive input validation, security audit, penetration testing

## Future Enhancements

### v2 Features
- Bulk operations (create multiple discussions)
- Advanced search and filtering
- Discussion templates
- Analytics and insights
- Webhook integration

### Integration Opportunities
- Connect discussions to issues/PRs
- Auto-create discussions from issues
- Discussion-based workflows
- AI-powered response suggestions

## Conclusion

Implementing GitHub Discussions support in the MCP Server will:
1. Fill a significant feature gap
2. Enable new automation workflows
3. Improve community engagement capabilities
4. Demonstrate MCP's flexibility and power

The proposed phased approach ensures:
- Incremental value delivery
- Comprehensive testing at each stage
- Time for community feedback
- Stable, production-ready implementation

**Estimated Timeline**: 4 weeks  
**Team Size**: 1-2 developers  
**Complexity**: Medium  
**Impact**: High  

This feature positions the GitHub MCP Server as the most comprehensive GitHub integration for AI assistants and automation workflows.
