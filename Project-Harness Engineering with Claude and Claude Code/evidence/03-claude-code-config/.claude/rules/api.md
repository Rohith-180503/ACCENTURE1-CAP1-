---
description: Standards and guidelines for Node.js API endpoint controllers
paths:
  - "src/api/**/*"
---
API Controller Conventions
Automatically applied whenever modifying code within src/api/.

Controller Signatures & Types

Controllers must conform to the signature:

async function handle(req: ApiRequest<TBody>): Promise<ApiResponse<TOut>>

Never invoke res.send() or directly manipulate response streams; the underlying server adapter manages HTTP outputs.

Infer generic parameters (TBody and TOut) strictly from defined schemas in src/api/_schemas/. Defining anonymous inline types is prohibited.

Error Propagation
Validation Issues: Raise a standard exception using throw new ApiError(400, "VALIDATION_FAILED", "..."). Do not return manual { status: 400 } payload objects; permit top-level error middleware to process thrown errors.

Missing Resources: Issue throw new ApiError(404, "NOT_FOUND", "..."). Returning null payloads like { status: 404, body: null } is disallowed.

Unhandled Exceptions: Allow runtime failures to bubble up naturally. Avoid catching, logging, and rethrowing unless adding critical context.

Data Access Layer
Perform database queries solely via repository modules in src/db/<table>.ts. Direct imports of pool, prisma, or ORM client instances inside controllers are forbidden.

Group multi-statement mutations inside a withTransaction(async (tx) => { ... }) block sourced from src/db/tx.ts.

Payload Verification
Execute request validation at the start of the function body using bodySchema.parse(req.body) prior to running business logic.

Once data passes schema validation, treat the parsed object as fully typed and verified downstream.

Structured Logging
Use the contextual logger bound to the request object: req.log.info({ field: value }, "message"). Raw console.log statements are forbidden anywhere in src/api/.

Logging Thresholds:

debug: Granular details during local debugging.

info: Key operational events during request processing.

warn: Non-fatal or auto-recovered edge cases.

error: Exclusively reserved for caught exceptions.