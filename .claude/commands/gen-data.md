# /gen-data — Generate Test Data Fixtures

Generates synthetic test data using `@faker-js/faker` and optional boundary/negative sets.
Produces TypeScript factory functions and static JSON data files for use in test specs.

## Arguments

| Argument | Description |
|---|---|
| `--schema <path>` | JSON schema file describing the data model (optional) |
| `--entities <list>` | Comma-separated entity names to generate (e.g., `user,product,order`) |
| `--count <n>` | Number of data rows per entity in JSON files (default: 5) |
| `--boundary` | Also generate boundary value sets (min/max/empty/null) |
| `--negative` | Also generate negative/invalid data sets (wrong formats, SQL injection, XSS) |

## Execution Steps

1. **Read kit.config.json** — get `outputDir`, `projectName`, `tech.language`

2. **Determine entities**:
   - If `--schema` provided: read schema → extract entity names and field types
   - If `--entities` provided: use entity names + infer field types from names (e.g., `email` → email format)
   - If neither: read `intake.summary.json` (if exists) and infer entities from test case steps
   - Fallback: generate a generic `user` entity with common fields

3. **Spawn datagen-agent** (read `tc-to-automate/kits/_base/agents/datagen-agent.md`)
   - Pass: `entities`, `count`, `boundary`, `negative`, `language`, `kitConfig`
   - Agent uses `data-factory` skill for faker patterns and boundary class generation

4. **Write artifacts**:
   - `outputs/<project>/src/fixtures/factories.ts` — factory functions
   - `outputs/<project>/src/fixtures/data/<entity>.valid.json` — n valid rows
   - `outputs/<project>/src/fixtures/data/<entity>.boundary.json` (if `--boundary`)
   - `outputs/<project>/src/fixtures/data/<entity>.invalid.json` (if `--negative`)

5. **Validate**: Run `npx tsc --noEmit` — factory file must compile

6. **Report**: Show factory function signatures and sample generated values

## Output Example

```typescript
// src/fixtures/factories.ts (generated)
import { faker } from '@faker-js/faker';

export const userFactory = () => ({
  email:     faker.internet.email(),
  password:  faker.internet.password({ length: 12, memorable: false }),
  firstName: faker.person.firstName(),
  lastName:  faker.person.lastName(),
});

// Usage in spec:
// const user = userFactory();
// await loginPage.login(user.email, user.password);
```

## Notes
- Factory functions are designed to be called fresh each test to avoid state sharing
- Boundary sets follow equivalence partitioning: empty string, 1 char, max-length+1, null, undefined
- Negative sets include: SQL injection (`' OR 1=1--`), XSS (`<script>alert(1)</script>`), malformed emails
- All data files are gitignored by default unless the user explicitly commits them
