# Sahmet-Org Enterprise Development Environment

A comprehensive microservices development platform featuring automated authentication, service mesh architecture, and integrated development tooling for enterprise-scale applications.

## Executive Summary
This development environment implements industry-standard practices for microservices architecture with:
- **Zero-trust security** via Keycloak OIDC authentication
- **API Gateway pattern** using Envoy for intelligent routing
- **Container-first development** with Docker Compose orchestration
- **Unified debugging** across all services with VS Code integration


## Architecture Overview
### Core Infrastructure
- **API Gateway**: Envoy proxy handling routing, authentication, and load balancing
- **Identity Provider**: Keycloak OIDC for centralized authentication and authorization
- **Service Mesh**: Container-based microservices with health monitoring
- **Development Platform**: VS Code multi-root workspace with integrated debugging

### Service Portfolio
| Service | Port | Purpose | Status |
|---------|------|---------|--------|
| Backend API | 8091 | Core business logic and data management | Active |
| Gateway | 9080 | API routing and authentication | Active |
| Auth Service | 8095 | Identity and access management | Active |
| Mail Service | 9999 | Development email testing | Active |


## Prerequisites
### Required Infrastructure
- **Docker Desktop**: Minimum 16GB memory allocation for production-grade performance
- **ngrok Account**: Public tunnel for development testing (free tier sufficient)
- **VS Code**: Latest version with recommended extensions
- **Python 3.10+**: Via pyenv for version management

### Access Requirements
- GitHub personal access token for private repository access
- AWS credentials for cloud resource integration (optional)
- Development environment admin access


## Quick Start
### 1. Environment Setup
```bash
# Clone the development environment
git clone https://github.com/your-org/sahmet-org.git
cd sahmet-org

# Open VS Code workspace (critical - don't open folder directly)
code sahmet-org.code-workspace
```

### 2. Infrastructure Bootstrap
```bash
# Start core services
docker compose up -d

# Create public tunnel
ngrok http 9080

# Update environment with your ngrok URL
# Edit .env file: NGROK_DOMAIN=your-hash.ngrok-free.app
```

### 3. Verification
Navigate to your ngrok URL to verify:
- `https://your-hash.ngrok-free.app/` - Frontend application
- `https://your-hash.ngrok-free.app/auth/admin/` - Keycloak admin (admin/password)
- `https://your-hash.ngrok-free.app/api/` - Backend API (protected)


## Development Workflow
### Service Development
Each service supports hot-reload development with integrated debugging:

```bash
# Start service in development mode
cd ./backend
docker compose up backend_dev

# Attach debugger in VS Code (port 5678)
# Set breakpoints and debug interactively
```

### Authentication Flow
1. User requests protected resource
2. Envoy gateway validates JWT token
3. If no valid token, redirect to Keycloak
4. Post-authentication, forward with JWT headers
5. Backend validates token via middleware

### Code Quality Standards
- **Black formatting**: Enforced via pre-commit hooks
- **Type checking**: Pylint with Django-specific rules
- **Testing**: Pytest with coverage requirements
- **Security**: Automated vulnerability scanning


## Security Architecture
### Zero-Trust Implementation
- All inter-service communication authenticated
- JWT tokens with configurable expiration
- OIDC-compliant identity verification
- Encrypted data transmission via TLS

### Development Security
- No plaintext secrets in repository
- Environment-specific configuration
- Container isolation for services
- Network segmentation via Docker networks


## Monitoring & Observability
### Health Monitoring
```bash
# Check service health
curl https://your-domain.ngrok-free.app/api/health/

# View container logs
docker compose logs -f backend_dev

# Monitor gateway routing
curl localhost:19000/stats
```

### Performance Metrics
- Response time monitoring via Envoy metrics
- Database query performance tracking
- Container resource utilization
- Authentication success rates


## Production Readiness
### Deployment Pipeline
- Multi-stage Docker builds for optimized images
- Environment promotion via GitOps workflow
- Blue-green deployment capability
- Automated rollback mechanisms

### Scalability Features
- Horizontal scaling via container orchestration
- Database read replicas supported
- CDN integration for static assets
- Auto-scaling based on metrics


## Troubleshooting
### Common Issues
**Authentication Failures**
```bash
# Verify Keycloak connectivity
curl https://your-domain.ngrok-free.app/auth/realms/myrealm/.well-known/openid_configuration

# Check JWT token validity
docker compose logs gateway | grep OIDC
```

**Service Communication Issues**
```bash
# Verify service registration
docker compose ps

# Test internal networking
docker compose exec backend ping keycloak.local
```

**Performance Issues**
```bash
# Monitor resource usage
docker stats

# Check database connections
docker compose exec db psql -U postgres -c "SELECT * FROM pg_stat_activity;"
```

### Support Escalation
1. Check service logs for errors
2. Verify environment configuration
3. Test connectivity between services
4. Consult architecture team for complex issues


## Contributing
### Development Standards
- Follow established coding conventions
- Maintain test coverage above 80%
- Update documentation for architectural changes
- Security review required for authentication changes

### Architecture Evolution
- Propose changes via ADR (Architecture Decision Records)
- Performance impact assessment required
- Backward compatibility maintained
- Migration path documented

---

**Version**: 1.0.2
**Last Updated**: Generated by Enterprise Wizard
**Support**: Development Team
