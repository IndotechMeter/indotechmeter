# Shared Types Implementation Plan

## Overview
This document outlines the strategy for implementing shared TypeScript types between backend and frontend to ensure type consistency across the IndoTech ERP system.

## Current State Analysis

### Problems Identified
1. **Type Duplication**: Same entities defined differently in backend/frontend
2. **Field Mismatches**:
   - Backend: `customerId` as primary key
   - Frontend: Both `id` and `customerId` fields
   - Optional handling: `string | null` vs `string | undefined`
3. **No Single Source of Truth**: Changes require updates in multiple places
4. **Type Drift**: Types gradually diverge over time

### Solution: Shared Types Package
Created `@indotech/shared-types` package at repository root with:
- Common type definitions
- Type guards for runtime validation
- Utility types for common patterns
- Constants for shared values

## Implementation Structure

```
indotech_erp/
├── shared-types/                 # New shared types package
│   ├── src/
│   │   ├── customer.types.ts    # Customer domain types
│   │   ├── api.types.ts         # API request/response types
│   │   ├── common.types.ts      # Common utility types
│   │   └── index.ts             # Main exports
│   ├── package.json
│   ├── tsconfig.json
│   └── README.md
├── backend/
│   └── indotech-erp-[service]/
│       └── package.json          # Add: "@indotech/shared-types": "file:../../shared-types"
└── frontend/
    └── package.json              # Add: "@indotech/shared-types": "file:../shared-types"
```

## Migration Steps

### Phase 1: Setup (Completed ✅)
1. ✅ Create shared-types package structure
2. ✅ Define Customer types as proof of concept
3. ✅ Add common utility types
4. ✅ Create API response types
5. ✅ Setup build configuration

### Phase 2: Integration (Next Steps)

#### Backend Integration
```bash
# For each backend service
cd backend/indotech-erp-customers
npm install ../../shared-types

# Update imports
# Before: import { Customer } from '../models/customer';
# After:  import { Customer } from '@indotech/shared-types';
```

#### Frontend Integration
```bash
cd frontend
npm install ../shared-types

# Update imports
# Before: import { Customer } from './types/customer.types';
# After:  import { Customer } from '@indotech/shared-types';
```

### Phase 3: Type Migration Priority

1. **Customer Types** (Week 1)
   - Most straightforward migration
   - Clear type structure
   - Used across multiple modules

2. **Invoice/Receivables Types** (Week 2)
   - Complex nested structures
   - Money type integration
   - Payment allocation logic

3. **Product Types** (Week 3)
   - Inventory integration
   - HSN codes and tax calculations
   - Category hierarchies

4. **HR/Payroll Types** (Week 4)
   - Employee records
   - Attendance/Leave structures
   - Compensation details

5. **Petty Cash Types** (Week 5)
   - Transaction types
   - Approval workflows
   - Balance calculations

## Type Categories in Shared Package

### 1. Domain Types
- Entity definitions (Customer, Invoice, Product)
- Business logic types (Money, Tax calculations)
- Enums and constants

### 2. API Types
- Request/Response wrappers
- Pagination structures
- Error formats
- HTTP utilities

### 3. Common Types
- Utility types (Nullable, DeepPartial)
- Date/Time helpers
- Audit fields
- User context

### 4. Infrastructure Types
- AWS Lambda event types
- DynamoDB structures
- S3 metadata
- Cognito claims

## Benefits

### Immediate Benefits
- **Type Safety**: Compile-time checking across boundaries
- **IDE Support**: Auto-completion everywhere
- **Refactoring**: Change once, update everywhere
- **Documentation**: Types as contracts

### Long-term Benefits
- **Reduced Bugs**: Catch mismatches early
- **Faster Development**: Less debugging type issues
- **Better Collaboration**: Clear contracts
- **Maintainability**: Single source of truth

## Best Practices

### DO ✅
- Keep types simple and focused
- Use type guards for runtime validation
- Document complex types with JSDoc
- Version carefully - breaking changes affect all
- Test type changes thoroughly

### DON'T ❌
- Don't duplicate types locally
- Don't use `any` to bypass type checks
- Don't mix concerns in type files
- Don't commit without building shared-types
- Don't use different patterns for similar types

## Common Patterns

### Optional Fields
```typescript
// Consistent pattern for optional fields
interface Entity {
  required: string;        // Always present
  optional?: string;       // May not be present
  nullable: string | null; // Present but can be null
}
```

### API Responses
```typescript
// Always wrap in ApiResponse
import { ApiResponse } from '@indotech/shared-types';

const response: ApiResponse<Customer> = {
  success: true,
  data: customer,
  message: 'Customer retrieved successfully'
};
```

### Money Handling
```typescript
// Never use raw numbers for money
import { Money } from '@indotech/shared-types';

const price: Money = {
  amount: 1000,
  currency: 'INR'
};
```

## Validation Strategy

### Compile-time
- TypeScript strict mode
- No implicit any
- Strict null checks

### Runtime
- Type guards at API boundaries
- Zod schemas derived from types
- Validation before database operations

## Testing Approach

### Type Testing
```typescript
// Test type compatibility
import { Customer } from '@indotech/shared-types';

const testCustomer: Customer = {
  customerId: 'test',
  // Ensures all required fields
};
```

### Integration Testing
- Test API contracts match types
- Validate database schemas align
- Check frontend can consume backend types

## Monitoring & Maintenance

### Version Management
- Semantic versioning for shared-types
- Coordinate updates across services
- Changelog for breaking changes

### Type Auditing
- Regular review for type drift
- Remove unused types
- Consolidate similar types

### Documentation
- Keep README updated
- Document migration guides
- Maintain type examples

## Risk Mitigation

### Potential Risks
1. **Breaking Changes**: Could affect multiple services
2. **Build Dependencies**: Shared package must build first
3. **Version Conflicts**: Services on different versions

### Mitigation Strategies
1. **Careful Versioning**: Major versions for breaking changes
2. **CI/CD Integration**: Build shared-types in pipeline
3. **Gradual Migration**: Service by service approach
4. **Backwards Compatibility**: Support old types temporarily

## Success Metrics

### Short-term (1 month)
- ✅ Shared-types package created
- [ ] Customer types migrated
- [ ] 50% reduction in type-related bugs
- [ ] All new types in shared package

### Medium-term (3 months)
- [ ] All services using shared types
- [ ] Zero type duplication
- [ ] Automated type validation
- [ ] Type coverage > 95%

### Long-term (6 months)
- [ ] Type-driven development standard
- [ ] API contracts auto-generated from types
- [ ] Frontend/Backend perfectly aligned
- [ ] Type changes require single update

## Next Immediate Actions

1. **Install Dependencies**
   ```bash
   cd shared-types && npm install
   ```

2. **Build Package**
   ```bash
   npm run build
   ```

3. **Update Backend Service** (Start with customers)
   ```bash
   cd backend/indotech-erp-customers
   npm install ../../shared-types
   # Update imports in code
   ```

4. **Update Frontend**
   ```bash
   cd frontend
   npm install ../shared-types
   # Update imports in components
   ```

5. **Test Integration**
   - Run backend tests
   - Run frontend build
   - Verify type checking works

## Conclusion

The shared types package provides a solid foundation for type consistency across the IndoTech ERP system. Starting with Customer types as a proof of concept, we can gradually migrate all shared types to this central package, ensuring type safety and reducing maintenance overhead.