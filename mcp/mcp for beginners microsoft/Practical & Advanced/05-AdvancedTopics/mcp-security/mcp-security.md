# ==Security Best Practices==

Security is critical for MCP implementations, especially in enterprise environments. It's important to ensure that tools and data are protected against(chống lại) ==unauthorized access, data breaches(rò rỉ), and other security threats.==

## Introduction

The Model Context Protocol (MCP) requires ==specific security considerations== due to its role in ==providing LLMs with access to data and tools==. This lesson explores ==security best practices== for MCP implementations based on official MCP guidelines and ==established security patterns.==

MCP follows key security principles to ensure safe and ==trustworthy interactions:==

- ==**User Consent and Control**==: ==Users must provide explicit consent before any data is accessed or operations are performed==. Provide clear control over ==what data is shared and which actions are authorized.==
  
- **==Data Privacy**==: ==User data should only be exposed with explicit consent and must be protected by appropriate access controls==. ==Safeguard against unauthorized data transmission.==
  
- ==**Tool Safety**==: ==Before invoking any tool, explicit user consent is required==. ==Users should have a clear understanding of each tool's functionality, and robust security boundaries must be enforced.==

## Learning Objectives

By the end of this lesson, you will be able to:

- Implement ==secure authentication and authorization mechanisms for MCP servers.==
- ==Protect sensitive data using encryption and secure storage.==
- ==Ensure secure execution of tools with proper access controls.==
- ==Apply best practices for data protection and privacy compliance.==
- ==Identify and mitigate common MCP security vulnerabilities like confused deputy problems, token passthrough, and session hijacking.==

## Authentication and Authorization

Authentication and authorization are essential for securing MCP servers. ==Authentication answers the question "Who are you?" while authorization answers "What can you do?".==

According to the MCP security specification, these are critical security considerations:

1. **Token Validation**: ==MCP servers MUST NOT accept any tokens that were not explicitly issued for the MCP server==. "Token passthrough" is an explicitly forbidden anti-pattern.

2. **Authorization Verification**: ==MCP servers that implement authorization MUST verify all inbound requests and MUST NOT use sessions for authentication.==

3. **Secure Session Management**: ==When using sessions for state, MCP servers MUST use secure, non-deterministic session IDs generated with secure random number generators==. ==Session IDs SHOULD be bound to user-specific information.==

4. **Explicit User Consent**: ==For proxy servers, MCP implementations MUST obtain user consent for each dynamically registered client before forwarding to third-party authorization servers.==

Let's look at examples of how to implement secure authentication and authorization in MCP servers using .NET and Java.

### .NET Identity Integration

ASP .NET Core Identity provides a robust framework for managing user authentication and authorization. We can integrate it with MCP servers to secure access to tools and resources.

There are some core concepts we need to understand when integrating ASP.NET Core Identity with MCP servers namely:

- **Identity Configuration**: Setting up ASP.NET Core Identity with user roles and claims. A claim is a piece of information about the user, such as their role or permissions for example "Admin" or "User".
- **JWT Authentication**: Using JSON Web Tokens (JWT) for secure API access. JWT is a standard for securely transmitting information between parties as a JSON object, which can be verified and trusted because it is digitally signed.
- **Authorization Policies**: ==Defining policies to control access to specific tools based on user roles.== MCP uses authorization policies to ==determine which users can access which tools based on their roles and claims.==

```csharp
public class SecureMcpStartup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Add ASP.NET Core Identity
        services.AddIdentity<ApplicationUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();
        
        // Configure JWT authentication
        services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
        })
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer = Configuration["Jwt:Issuer"],
                ValidAudience = Configuration["Jwt:Audience"],
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(Configuration["Jwt:Key"]))
            };
        });
        
        // Add authorization policies
        services.AddAuthorization(options =>
        {
            options.AddPolicy("CanUseAdminTools", policy =>
                policy.RequireRole("Admin"));
                
            options.AddPolicy("CanUseBasicTools", policy =>
                policy.RequireAuthenticatedUser());
        });
        
        // Configure MCP server with security
        services.AddMcpServer(options =>
        {
            options.ServerName = "Secure MCP Server";
            options.ServerVersion = "1.0.0";
            options.RequireAuthentication = true;
        });
        
        // Register tools with authorization requirements
        services.AddMcpTool<BasicTool>(options => 
            options.RequirePolicy("CanUseBasicTools"));
            
        services.AddMcpTool<AdminTool>(options => 
            options.RequirePolicy("CanUseAdminTools"));
    }
    
    public void Configure(IApplicationBuilder app)
    {
        // Use authentication and authorization
        app.UseAuthentication();
        app.UseAuthorization();
        
        // Use MCP server middleware
        app.UseMcpServer();
    }
}
```

In the preceding code, we have:

- Configured ASP.NET Core Identity for user management.
- Set up JWT authentication for secure API access. We specified the token validation parameters, including the issuer, audience, and signing key.
- ==Defined authorization policies to control access to tools based on user roles. For example, the "CanUseAdminTools" policy requires the user to have the "Admin" role, while the "CanUseBasic" policy requires the user to be authenticated.==
- Registered MCP tools with specific authorization requirements, ensuring that only users with the appropriate roles can access them. 

### ==Java Spring Security Integration==

For Java, we will use Spring Security to implement secure authentication and authorization for MCP servers. Spring Security provides a comprehensive security framework that integrates seamlessly with Spring applications.


Core concepts here are:

- **Spring Security Configuration**: Setting up security configurations for authentication and authorization.
- **OAuth2 Resource Server**: Using OAuth2 for secure access to MCP tools. OAuth2 is an authorization framework that allows third-party services to exchange access tokens for secure API access.
- **Security Interceptors**: Implementing security interceptors to enforce access controls on tool execution.
- **Role-Based Access Control**: Using roles to control access to specific tools and resources.
- **Security Annotations**: Using annotations to secure methods and endpoints.


```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeRequests()
                .antMatchers("/mcp/discovery").permitAll() // Allow tool discovery
                .antMatchers("/mcp/tools/**").hasAnyRole("USER", "ADMIN") // Require authentication for tools
                .antMatchers("/mcp/admin/**").hasRole("ADMIN") // Admin-only endpoints
                .anyRequest().authenticated()
            .and()
            .oauth2ResourceServer().jwt();
    }
    
    @Bean
    public McpSecurityInterceptor mcpSecurityInterceptor() {
        return new McpSecurityInterceptor();
    }
}

// MCP Security Interceptor for tool authorization
public class McpSecurityInterceptor implements ToolExecutionInterceptor {
    @Autowired
    private JwtDecoder jwtDecoder;
    
    @Override
    public void beforeToolExecution(ToolRequest request, Authentication authentication) {
        String toolName = request.getToolName();
        
        // Check if user has permissions for this tool
        if (toolName.startsWith("admin") && !authentication.getAuthorities().contains("ROLE_ADMIN")) {
            throw new AccessDeniedException("You don't have permission to use this tool");
        }
        
        // Additional security checks based on tool or parameters
        if ("sensitiveDataAccess".equals(toolName)) {
            validateDataAccessPermissions(request, authentication);
        }
    }
    
    private void validateDataAccessPermissions(ToolRequest request, Authentication auth) {
        // Implementation to check fine-grained data access permissions
    }
}
```

In the preceding code, we have:

- Configured Spring Security to secure MCP endpoints, ==allowing public access to tool discovery while requiring authentication for tool execution.==
- ==Used OAuth2 as a resource server to handle secure access to MCP tools.==
- Implemented a ==security interceptor== to ==enforce access controls on tool execution==, c==hecking user roles and permissions before allowing access to specific tools.==
- ==Defined role-based access control to restrict access to admin tools and sensitive data access based on user roles.==

## ==Data Protection and Privacy==

Data protection is ==crucial for ensuring that sensitive information is handled securely==. This includes ==protecting personally identifiable information== (PII), ==financial data==, and ==other sensitive information from unauthorized access and breaches.==

### Python Data Protection Example

Let's look at an example of how to implement data protection in Python using encryption and PII detection.

```python
from mcp_server import McpServer
from mcp_tools import Tool, ToolRequest, ToolResponse
from cryptography.fernet import Fernet
import os
import json
from functools import wraps

# PII Detector - identifies and protects sensitive information
class PiiDetector:
    def __init__(self):
        # Load patterns for different types of PII
        with open("pii_patterns.json", "r") as f:
            self.patterns = json.load(f)
    
    def scan_text(self, text):
        """Scans text for PII and returns detected PII types"""
        detected_pii = []
        # Implementation to detect PII using regex or ML models
        return detected_pii
    
    def scan_parameters(self, parameters):
        """Scans request parameters for PII"""
        detected_pii = []
        for key, value in parameters.items():
            if isinstance(value, str):
                pii_in_value = self.scan_text(value)
                if pii_in_value:
                    detected_pii.append((key, pii_in_value))
        return detected_pii

# Encryption Service for protecting sensitive data
class EncryptionService:
    def __init__(self, key_path=None):
        if key_path and os.path.exists(key_path):
            with open(key_path, "rb") as key_file:
                self.key = key_file.read()
        else:
            self.key = Fernet.generate_key()
            if key_path:
                with open(key_path, "wb") as key_file:
                    key_file.write(self.key)
        
        self.cipher = Fernet(self.key)
    
    def encrypt(self, data):
        """Encrypt data"""
        if isinstance(data, str):
            return self.cipher.encrypt(data.encode()).decode()
        else:
            return self.cipher.encrypt(json.dumps(data).encode()).decode()
    
    def decrypt(self, encrypted_data):
        """Decrypt data"""
        if encrypted_data is None:
            return None
        
        decrypted = self.cipher.decrypt(encrypted_data.encode())
        try:
            return json.loads(decrypted)
        except:
            return decrypted.decode()

# Security decorator for tools
def secure_tool(requires_encryption=False, log_access=True):
    def decorator(cls):
        original_execute = cls.execute_async if hasattr(cls, 'execute_async') else cls.execute
        
        @wraps(original_execute)
        async def secure_execute(self, request):
            # Check for PII in request
            pii_detector = PiiDetector()
            pii_found = pii_detector.scan_parameters(request.parameters)
            
            # Log access if required
            if log_access:
                tool_name = self.get_name()
                user_id = request.context.get("user_id", "anonymous")
                log_entry = {
                    "timestamp": datetime.now().isoformat(),
                    "tool": tool_name,
                    "user": user_id,
                    "contains_pii": bool(pii_found),
                    "parameters": {k: "***" for k in request.parameters.keys()}  # Don't log actual values
                }
                logging.info(f"Tool access: {json.dumps(log_entry)}")
            
            # Handle detected PII
            if pii_found:
                # Either encrypt sensitive data or reject the request
                if requires_encryption:
                    encryption_service = EncryptionService("keys/tool_key.key")
                    for param_name, pii_types in pii_found:
                        # Encrypt the sensitive parameter
                        request.parameters[param_name] = encryption_service.encrypt(
                            request.parameters[param_name]
                        )
                else:
                    # If encryption not available but PII found, you might reject the request
                    raise ToolExecutionException(
                        "Request contains sensitive data that cannot be processed securely"
                    )
            
            # Execute the original method
            return await original_execute(self, request)
        
        # Replace the execute method
        if hasattr(cls, 'execute_async'):
            cls.execute_async = secure_execute
        else:
            cls.execute = secure_execute
        return cls
    
    return decorator

# Example of a secure tool with the decorator
@secure_tool(requires_encryption=True, log_access=True)
class SecureCustomerDataTool(Tool):
    def get_name(self):
        return "customerData"
    
    def get_description(self):
        return "Accesses customer data securely"
    
    def get_schema(self):
        # Schema definition
        return {}
    
    async def execute_async(self, request):
        # Implementation would access customer data securely
        # Since we used the decorator, PII is already detected and encrypted
        return ToolResponse(result={"status": "success"})
```

In the preceding code, we have:

- Implemented a `PiiDetector` class to ==scan text and parameters for personally identifiable information (==PII).
- Created an `EncryptionService` class to ==handle encryption and decryption of sensitive data using the `cryptography` library.==
- Defined a `secure_tool` ==decorator that wraps tool execution to check for PII, log access, and encrypt sensitive data if required.==
- Applied the `secure_tool` ==decorator to a sample tool (`SecureCustomerDataTool`) to ensure it handles sensitive data securely.==

**Request gốc:**

python

```python
request = ToolRequest(
    tool="customerData",
    parameters={
        "customer_id": "12345",
        "email": "user@example.com",  # PII!
        "action": "get_profile"
    }
)
```

**Sau khi qua decorator:**

python

```python
# 1. PII Detected: email field
# 2. Log entry created:
{
    "timestamp": "2024-01-20T10:30:00",
    "tool": "customerData",
    "user": "admin123",
    "contains_pii": true,
    "parameters": {
        "customer_id": "***",
        "email": "***",
        "action": "***"
    }
}

# 3. Parameters encrypted:
{
    "customer_id": "12345",  # Không phải PII
    "email": "gAAAAABhZ7x...",  # Đã mã hóa
    "action": "get_profile"
}
```
## MCP-Specific Security Risks

According to the official MCP security documentation, there are specific security risks that MCP implementers should be aware of:

### 1. Confused Deputy Problem

This vulnerability occurs when an MCP server acts as a proxy to third-party APIs, potentially allowing attackers to exploit the trusted relationship between the MCP server and these APIs.

**Mitigation:**
- MCP proxy servers using static client IDs MUST obtain user consent for each dynamically registered client before forwarding to third-party authorization servers.
- Implement proper OAuth flow with PKCE (Proof Key for Code Exchange) for authorization requests.
- Strictly validate redirect URIs and client identifiers.

### 2. Token Passthrough Vulnerabilities

Token passthrough occurs when an MCP server accepts tokens from an MCP client without validating that the tokens were properly issued to the MCP server and passes them through to downstream APIs.

### Risks
- Security control circumvention (bypassing rate limiting, request validation)
- Accountability and audit trail issues
- Trust boundary violations
- Future compatibility risks

**Mitigation:**
- MCP servers MUST NOT accept any tokens that were not explicitly issued for the MCP server.
- Always validate token audience claims to ensure they match the expected service.

### 3. Session Hijacking

This occurs when an unauthorized party obtains a session ID and uses it to impersonate the original client, potentially leading to unauthorized actions.

**Mitigation:**
- MCP servers that implement authorization MUST verify all inbound requests and MUST NOT use sessions for authentication.
- Use secure, non-deterministic session IDs generated with secure random number generators.
- Bind session IDs to user-specific information using a key format like `<user_id>:<session_id>`.
- Implement proper session expiration and rotation policies.

## Additional Security Best Practices for MCP

Beyond the core MCP security considerations, consider implementing these additional security practices:

- **Always use HTTPS**: Encrypt communication between client and server to protect tokens from interception.
- **Implement Role-Based Access Control (RBAC)**: Don't just check if a user is authenticated; check what they are authorized to do.
- **Monitor and audit**: Log all authentication events to detect and respond to suspicious activity.
- **Handle rate limiting and throttling**: Implement exponential backoff and retry logic to handle rate limits gracefully.
- **Secure token storage**: Store access tokens and refresh tokens securely using system secure storage mechanisms or secure key management services.
- **Consider using API Management**: Services like Azure API Management can handle many security concerns automatically, including authentication, authorization, and rate limiting.

## References

- [MCP Security Best Practices](https://modelcontextprotocol.io/specification/draft/basic/security_best_practices)
- [MCP Authorization Specification](https://modelcontextprotocol.io/specification/draft/basic/authorization)
- [MCP Core Concepts](https://modelcontextprotocol.io/docs/concepts/architecture)
- [OAuth 2.0 Security Best Practices (RFC 9700)](https://datatracker.ietf.org/doc/html/rfc9700)

## What's next

- [5.9 Web search](../web-search-mcp/README.md)