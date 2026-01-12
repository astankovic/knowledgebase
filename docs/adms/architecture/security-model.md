---
layout: default
title: Security Model
parent: Architecture
grand_parent: ADMS Overview
nav_order: 5
---

# Security Model

## Overview

Security is fundamental to ADMS architecture given the system's role in controlling critical electrical infrastructure. The security model implements defense-in-depth principles with multiple layers of protection including authentication, authorization, encryption, network security, and audit logging. The system must protect against unauthorized access, data breaches, and cyber attacks while maintaining compliance with utility industry regulations including NERC CIP (Critical Infrastructure Protection) standards.

Modern ADMS deployments face sophisticated threat actors targeting critical infrastructure. Security architecture must address external threats from the internet, insider threats from malicious or negligent users, and supply chain risks from third-party integrations. The system implements zero-trust principles where no user or service is inherently trusted, and every access request is authenticated, authorized, and encrypted regardless of network location.

Security controls balance protection with operational usability. Overly restrictive controls can impede operators during emergency situations when rapid response is critical. The security model enables appropriate access based on user roles, context, and risk levels while maintaining comprehensive audit trails for compliance and forensic analysis.

## Authentication

### OAuth 2.0 and OpenID Connect

ADMS employs OAuth 2.0 and OpenID Connect for modern, standards-based authentication. Users authenticate through an identity provider (such as Azure Active Directory, Okta, or Keycloak) which issues JSON Web Tokens (JWT) upon successful authentication. These tokens contain user identity, roles, and expiration times, signed cryptographically to prevent tampering.

The authorization code flow with PKCE (Proof Key for Code Exchange) provides secure authentication for web applications. Users are redirected to the identity provider for authentication, receive an authorization code, which the application exchanges for access and refresh tokens. This flow prevents token exposure in browser history or logs.

Refresh tokens enable token renewal without requiring users to re-authenticate frequently. Access tokens have short lifetimes (15 minutes) to limit exposure if compromised, while refresh tokens have longer validity (hours to days) but are stored securely and can be revoked. Token refresh occurs transparently in background, maintaining user sessions without interruption.

### Multi-Factor Authentication

Critical operations and privileged access require multi-factor authentication (MFA) providing additional security beyond passwords. MFA combines something the user knows (password), something they have (mobile device, hardware token), and optionally something they are (biometric). Time-based one-time passwords (TOTP) or push notifications to mobile apps provide the second factor.

Administrative access to ADMS configuration and system functions always requires MFA. Control operations affecting physical equipment may require MFA or additional confirmation steps based on risk assessment. Step-up authentication challenges users for additional factors when performing sensitive operations even within an active session.

### LDAP and Active Directory Integration

For enterprises with existing directory services, ADMS integrates with LDAP or Active Directory for centralized user management. The identity provider federates authentication to these systems, enabling single sign-on across enterprise applications. Group memberships in Active Directory map to ADMS roles, centralizing access control management.

Service-to-service authentication uses OAuth client credentials flow where services authenticate with client ID and secret to obtain access tokens. Alternatively, mutual TLS (mTLS) authentication uses client certificates to authenticate services, providing cryptographic assurance of service identity without shared secrets.

## Authorization

### Role-Based Access Control (RBAC)

RBAC implements the principle of least privilege where users receive only the permissions necessary for their job functions. Roles are defined based on operational responsibilities: Operator, Engineer, Administrator, Read-Only User. Each role has associated permissions for system functions and data access.

Permissions are granular, controlling access to specific operations: view network model, execute calculations, issue control commands, modify configuration, view audit logs. The system evaluates user roles and permissions for every operation, denying unauthorized requests. Permission checks occur at multiple layers: API gateway, service layer, and database row-level security.

Role hierarchies enable permission inheritance where senior roles automatically include junior role permissions. The Administrator role inherits all Operator permissions plus administrative functions. This simplifies role management while maintaining clear permission boundaries.

### Contextual Authorization

Beyond static role permissions, contextual authorization considers additional factors: time of day (restricting access outside normal hours), geographic location (blocking access from unexpected countries), device posture (requiring managed devices), and recent authentication freshness (requiring re-authentication for sensitive operations).

Operational context affects authorization decisions. During emergency situations, expanded permissions may be granted temporarily to additional users. Pre-defined emergency procedures can be activated, modifying normal authorization rules for incident response while maintaining audit trails of expanded access.

### Resource-Level Permissions

Permissions extend to individual resources with fine-grained access control. Users may have permissions for specific geographic regions (substations, feeders) but not others. Engineers responsible for particular equipment can view and control only their assigned assets. This geographic or organizational partitioning limits potential damage from compromised accounts.

## Network Security

### TLS/SSL Encryption

All network communication uses TLS 1.2 or higher for encryption in transit. Client-to-server HTTPS connections protect user credentials and operational data from eavesdropping. Service-to-service communication within the system uses TLS even on internal networks, implementing zero-trust networking principles.

TLS configurations enforce strong cipher suites, disabling weak algorithms vulnerable to cryptanalysis. Forward secrecy cipher suites (ECDHE) ensure that compromise of long-term keys doesn't enable decryption of past communications. Certificate management includes regular rotation, revocation checking via OCSP (Online Certificate Status Protocol), and certificate pinning for critical connections.

### Firewall Configuration

Network segmentation isolates ADMS components based on security zones. DMZ networks host public-facing components (web servers, API gateways) with strict ingress filtering. Internal application and database networks permit only required service communication. SCADA networks are isolated from corporate IT networks, with controlled gateways enabling necessary data exchange.

Firewall rules implement whitelisting where only explicitly permitted traffic is allowed, with all other traffic denied by default. Rules specify source and destination IP addresses, ports, and protocols. Stateful firewalls track connection state, permitting return traffic for established connections while blocking unsolicited inbound traffic.

### VPN Access

Remote access to ADMS for engineers and vendors requires VPN connections providing encrypted tunnels over the internet. VPN configurations enforce strong authentication (certificate-based or MFA), encrypt all traffic, and route connections through firewall ingress points with logging. Split-tunnel configurations are avoided for accessing critical systems, ensuring all traffic traverses security controls.

## Data Protection

### Encryption at Rest

Sensitive data is encrypted at rest in databases and file systems. Database encryption (Transparent Data Encryption in PostgreSQL) encrypts data files at the storage layer. File system encryption (LUKS on Linux, BitLocker on Windows) protects against unauthorized access to storage media.

Encryption keys are managed through dedicated Key Management Systems (KMS) or Hardware Security Modules (HSM) providing tamper-resistant key storage. Keys are rotated periodically per policy. Encryption key access is strictly controlled and audited, with keys never exposed in application code or configuration files.

### Data Masking

Non-production environments (development, test) use masked or synthetic data to prevent exposure of production information. Data masking techniques replace sensitive values (names, locations, account numbers) with realistic but fictitious data. This enables testing with representative data without security risks of production data exposure.

## Audit and Compliance

### Comprehensive Audit Logging

All security-relevant events are logged: authentication attempts (success and failure), authorization decisions, configuration changes, control operations, data access, and administrative actions. Audit logs include timestamp, user identity, source IP address, action performed, and outcome.

Logs are centralized in tamper-evident log management systems with append-only access. Log integrity is protected through cryptographic hashing or signing. Logs are retained per compliance requirements (typically 3-7 years) with long-term archive to immutable storage.

### NERC CIP Compliance

Utilities in North America must comply with NERC CIP standards for critical cyber assets. Requirements include access control, security management, incident reporting, recovery planning, and personnel training. ADMS implementations include technical and procedural controls mapped to specific CIP requirements.

Compliance evidence is generated automatically: access logs for CIP-004, change logs for CIP-010, security event monitoring for CIP-008. Regular audits verify controls effectiveness and identify gaps requiring remediation.

## Code Examples

### JWT Validation

```java
@Component
public class JwtTokenValidator {

    @Value("${jwt.public-key}")
    private String publicKey;

    public Claims validateToken(String token) {
        try {
            return Jwts.parserBuilder()
                .setSigningKey(getPublicKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
        } catch (JwtException e) {
            throw new UnauthorizedException("Invalid token", e);
        }
    }

    public boolean hasRole(Claims claims, String role) {
        List<String> roles = claims.get("roles", List.class);
        return roles != null && roles.contains(role);
    }

    private PublicKey getPublicKey() {
        // Load public key from configuration
        byte[] keyBytes = Base64.getDecoder().decode(publicKey);
        X509EncodedKeySpec spec = new X509EncodedKeySpec(keyBytes);
        KeyFactory kf = KeyFactory.getInstance("RSA");
        return kf.generatePublic(spec);
    }
}
```

### Permission Check

```java
@Service
public class AuthorizationService {

    public void checkPermission(User user, String operation, Resource resource) {
        if (!hasPermission(user, operation, resource)) {
            auditLog.logAccessDenied(user, operation, resource);
            throw new ForbiddenException(
                String.format("User %s lacks permission for %s on %s",
                    user.getId(), operation, resource.getId())
            );
        }
        auditLog.logAccessGranted(user, operation, resource);
    }

    private boolean hasPermission(User user, String operation, Resource resource) {
        // Check role-based permissions
        for (Role role : user.getRoles()) {
            if (role.hasPermission(operation)) {
                // Check resource-level access
                if (canAccessResource(user, resource)) {
                    return true;
                }
            }
        }
        return false;
    }

    private boolean canAccessResource(User user, Resource resource) {
        // Check geographic/organizational restrictions
        return user.getAuthorizedRegions().contains(resource.getRegion());
    }
}
```

### Audit Logging

```java
@Aspect
@Component
public class AuditAspect {

    @Autowired
    private AuditLogger auditLogger;

    @Around("@annotation(Audited)")
    public Object auditOperation(ProceedingJoinPoint joinPoint) throws Throwable {
        String user = SecurityContextHolder.getContext().getAuthentication().getName();
        String operation = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();

        AuditEvent event = AuditEvent.builder()
            .timestamp(Instant.now())
            .user(user)
            .operation(operation)
            .arguments(args)
            .build();

        try {
            Object result = joinPoint.proceed();
            event.setOutcome("SUCCESS");
            return result;
        } catch (Exception e) {
            event.setOutcome("FAILURE");
            event.setError(e.getMessage());
            throw e;
        } finally {
            auditLogger.log(event);
        }
    }
}
```

## Best Practices

- Implement defense-in-depth with multiple security layers rather than relying on single controls
- Use short-lived access tokens (15 minutes) with automatic refresh to limit exposure if compromised
- Require MFA for administrative access and critical control operations
- Encrypt all network traffic with TLS 1.2+ even on internal networks
- Apply principle of least privilege with role-based access control and resource-level permissions
- Maintain comprehensive audit logs for all security-relevant events with tamper-evident storage
- Regularly rotate encryption keys, certificates, and credentials per security policy
- Conduct periodic security assessments including penetration testing and vulnerability scanning
- Implement security monitoring with automated alerting for suspicious activity
- Maintain incident response procedures and conduct regular tabletop exercises
