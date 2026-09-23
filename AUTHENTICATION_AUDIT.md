# Authentication and tenant audit

Public Core 0.4 supplies LocalTokenValidator through TokenValidatorConfig, configured with idax.auth.mode=LOCAL and idax.auth.token-validator=local. Signature verification uses the public half of the locally generated RSA pair at /run/secrets/jwt_public_key; Shell signs with the private half. Private key never enters STIR.

The initial adapter called TokenValidator directly, populated CurrentUser and delegated TenantContextFilter. The live superuser catalogs request returned 403. Public Core's migration comments and Shell composition establish that JwtAuthFilter is responsible for path/header/JWT tenant selection before TenantContextFilter. The corrected StirJwtAuthFilter delegates this public filter; it does not parse claims, implement signatures, select a tenant or query memberships itself. It only rejects validated service identities on personal endpoints and clears contexts. Servlet auto-registration is disabled to avoid duplicate invocation.

ListingController independently requires path tenantId == TenantContext. ListingService gets tenant from TenantContext and owner exclusively from CurrentUser. Neither body nor client ownership headers can assign owner. Unknown JSON fields are rejected. PermissionService checks read/create/update, then ownership checks constrain mutations even for administrators.

Final live invalid-token, contradictory-header, membership and A/B tests are recorded in VALIDATION.md. A filter unit test separately demonstrates rejection of a validated CurrentUser marked as service; it is not presented as real service-token provisioning through an unavailable endpoint.

Database runtime uses idax_backend with NOSUPERUSER/NOBYPASSRLS, no schema ownership or CREATE, and Core's idax_app/idax_admin role memberships. RlsTransactionAspect/DbSessionContextService establish transaction-local tenant/role. STIR's forced policy restricts both roles to app.tenant_id; it does not include an unrestricted admin policy. One-shot migration/provision jobs have separate credentials that runtime services never mount.

## Executed runtime proof

The HTTP suite provisions two isolated tenants through the public Core `idax_core.tenant_create` capability (the public Shell has no tenant-creation HTTP endpoint). It never inserts/updates Core tables. Ordinary local users are created through Shell HTTP; role creation, role permission assignment and user-role assignment use their separate public endpoints. Each receives only stir.listings.read/create/update. Authentication is repeated after assignment.

Both users read and update their own listing. Foreign UUID read/update/close returns 404 inside the authorized tenant; foreign tenant paths and contradictory X-Tenant headers are rejected. List and combined category/resourceKind/status/text filters return only the active tenant's listing. Owner headers have no effect and an ownerId in JSON is rejected.

Runtime SQL evidence reports idax_backend; rolsuper=false; rolbypassrls=false; schema/database CREATE=false. Granted roles are idax_app, idax_admin and idax_service_auth, as provided by public Core. Live JDBC sessions use idax_backend. The listing policy is enabled and forced, applies to idax_app/idax_admin, and uses the same tenant equality for USING and WITH CHECK. With no tenant the runtime sees zero rows; with transaction-local tenant A both roles see A and not B. Catalog tables contain only shared, read-only category/resource-kind codes, not tenant business data.

These checks demonstrate RLS in the deployed runtime in addition to Testcontainers. They do not claim RLS authenticates arbitrary SQL clients: Core establishes the trusted request context, and database access remains a privileged deployment boundary.
