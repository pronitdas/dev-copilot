# Testing Standards

## Testing Pyramid

All code must have tests at multiple levels:

- **Unit Tests**: Fast, focused, testing business logic in isolation
- **Integration Tests**: Validate service interactions with real dependencies
- **Contract Tests**: Ensure API compatibility between producer/consumer
- **Performance Tests**: Benchmarks with clear pass/fail criteria
- **Chaos Testing**: Regular disruption of services to ensure resilience
- **E2E Tests**: Critical user journeys from user perspective

## Coverage Requirements

- Minimum code coverage: 85%
- Critical paths must have 100% coverage
- All error conditions must be tested
- No untested code reaches production

## Unit Test Pattern

```typescript
describe('ServiceName', () => {
  let service: ServiceName;
  let mockDependency: jest.Mocked<Dependency>;

  beforeEach(() => {
    mockDependency = { /* mock methods */ } as jest.Mocked<Dependency>;
    service = new ServiceName(mockDependency);
  });

  describe('methodName', () => {
    it('should handle happy path', async () => {
      // Arrange
      const input = { /* valid input */ };
      mockDependency.method.mockResolvedValue({ /* expected output */ });

      // Act
      const result = await service.methodName(input);

      // Assert
      expect(mockDependency.method).toHaveBeenCalledWith(input);
      expect(result).toEqual(/* expected result */);
    });

    it('should throw on invalid input', async () => {
      // Arrange
      const invalidInput = { /* invalid */ };

      // Act & Assert
      await expect(service.methodName(invalidInput))
        .rejects.toThrow(/error message/);
    });
  });
});
```

## Integration Testing

For integration tests with databases:

- Use test containers or ephemeral databases
- Clean state before each test
- Test actual database interactions
- Verify transactions and rollbacks

## Performance Testing

- Define baseline metrics
- Set acceptable thresholds
- Test under realistic load
- Monitor resource consumption

## Enforcement

- Automated PR checks for test coverage
- Performance regression testing
- Failed tests block deployment
- Test metrics tracked per developer
