---
name: review
description: Guidelines for reviewing code changes, features, and pull requests.
---

# Reviewing Features and Pull Requests

Guidelines for reviewing code changes, features, and pull requests.

## Review Mindset

- Be thorough but constructive
- Focus on correctness, security, and maintainability
- Consider the user experience impact
- Look for both what's present and what's missing

## Review Checklist

### 1. Understand the Context

- [ ] Read the PR description and linked issues
- [ ] Understand the problem being solved
- [ ] Check if the approach aligns with project architecture

### 2. Code Quality

- [ ] Code is readable and follows project conventions
- [ ] Functions and variables have clear, descriptive names
- [ ] No unnecessary complexity or over-engineering
- [ ] Dead code and debug statements removed
- [ ] TypeScript types are properly defined (no `any` unless justified)

### 3. Functionality

- [ ] Code does what it claims to do
- [ ] Edge cases are handled (empty states, errors, null values)
- [ ] Loading states are shown during async operations
- [ ] Error states provide useful feedback to users
- [ ] Success states are clear and confirmed

### 4. Security

Review for common vulnerabilities:

- [ ] **Input validation** - User input is validated/sanitized
- [ ] **Authentication** - Protected routes check auth status
- [ ] **Authorization** - Users can only access their own data
- [ ] **No secrets exposed** - API keys, tokens not in client code
- [ ] **XSS prevention** - User content is properly escaped
- [ ] **CSRF protection** - State-changing requests are protected

For the API:
- [ ] Rate limiting on sensitive endpoints
- [ ] Proper error messages (don't leak internal details)
- [ ] Signed URLs for file access (no direct R2 keys exposed)

### 5. Performance

- [ ] No unnecessary re-renders or API calls
- [ ] Large lists are virtualized or paginated
- [ ] Images and assets are optimized
- [ ] Expensive operations are debounced/throttled
- [ ] No memory leaks (event listeners cleaned up)

### 6. User Experience

- [ ] UI matches existing design patterns
- [ ] Interactive elements have proper feedback (hover, focus, active states)
- [ ] Keyboard navigation works
- [ ] Screen reader accessibility considered
- [ ] Mobile responsiveness tested
- [ ] Loading and empty states are polished

### 7. API Integration

- [ ] API calls handle all response states (success, error, loading)
- [ ] Error messages are user-friendly
- [ ] Retry logic for transient failures where appropriate
- [ ] Optimistic updates rolled back on failure

### 8. Testing Considerations

- [ ] Happy path works as expected
- [ ] Error scenarios handled gracefully
- [ ] Boundary conditions tested (empty, max limits)
- [ ] Cross-browser compatibility if UI changes

## Common Issues to Flag

### Blockers (Must Fix)

- Security vulnerabilities
- Data loss or corruption risks
- Breaking changes without migration
- Missing error handling that crashes the app
- Authentication/authorization bypasses

### Should Fix

- Inconsistent code style
- Missing TypeScript types
- Poor variable/function names
- Duplicated code that should be shared
- Missing loading/error states

### Nice to Have

- Minor performance optimizations
- Additional edge case handling
- Code comments for complex logic
- Refactoring suggestions

## Giving Feedback

### Be Specific

```
// Vague
"This could be better"

// Specific
"Consider using `useSignal` here instead of `useState` since this
value is updated frequently in the drag handler"
```

### Explain the Why

```
// Just what
"Add error handling here"

// With why
"Add error handling here - if the API call fails, the user
currently sees a blank screen with no way to retry"
```

### Suggest Solutions

```
// Problem only
"This will cause unnecessary re-renders"

// With solution
"This will cause unnecessary re-renders. Consider memoizing
the callback with useCallback or moving the handler definition
outside the render"
```

### Use Appropriate Severity

- **Blocking**: "This must be fixed before merge because..."
- **Suggestion**: "Consider doing X instead..."
- **Question**: "I'm curious why you chose X over Y here?"
- **Nitpick**: "Nit: minor style preference..."

## Review Process

1. **First pass**: Skim for overall approach and architecture
2. **Second pass**: Detailed line-by-line review
3. **Third pass**: Test the feature manually if possible
4. **Final check**: Ensure all comments are addressed before approving

## When to Approve

Approve when:
- All blocking issues are resolved
- The code works as intended
- You're confident it won't introduce regressions
- Remaining comments are truly optional

Request changes when:
- There are security concerns
- Core functionality is broken
- The approach needs fundamental rethinking