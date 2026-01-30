# Precision Standards

## Core Requirement

**Schema contracts are law, not guidelines.**

- Every API request and response must validate against its schema
- No field should ever be "interpreted" — it either meets the contract or fails
- Changes to schemas require versioning and explicit migration paths
- Undefined behavior is a bug, not a feature

## Validation Implementation

Use validation at all boundaries:

```typescript
// Runtime validation at API boundaries
const CreateMarkerSchema = z.object({
  name: z.string().min(1).max(100),
  position: z.object({
    latitude: z.number().min(-90).max(90),
    longitude: z.number().min(-180).max(180),
  }),
});

app.post('/markers', validate(CreateMarkerSchema), async (req, res) => {
  // Now req.body is fully typed and validated
  const marker = await markerService.create(req.body);
  res.status(201).json(marker);
});
```

## Strict Type Safety

- Never use `as any` to bypass TypeScript
- Never use `@ts-ignore` or `@ts-expect-error`
- Use discriminated unions for error states
- Prefer `satisfies` over type assertions

## Error Handling

- Errors have explicit types and codes
- Error responses follow consistent schema
- Client errors (4xx) vs server errors (5xx) clearly distinguished
- Error messages never leak sensitive information

## Schema Evolution

1. **Backward-compatible changes** (add optional fields): Deploy immediately
2. **Breaking changes** (remove/rename fields): Version the API, support old version
3. **Deprecation**: Clear timeline, communicate to consumers
4. **Migration**: Provide upgrade path, test migrations

## Enforcement

- PR checks automatically verify schema compliance
- Schema regression tests run on every commit
- Schema violations in production trigger alerts
- Type checking enforced in CI pipeline
