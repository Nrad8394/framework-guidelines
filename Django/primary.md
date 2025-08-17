# Django DRF Production-Ready Development Guide

*Building High-Performance, Scalable APIs from Day One*

## Table of Contents
1. [Project Architecture](#project-architecture)
2. [Development Environment](#development-environment)
3. [Database & Models](#database--models)
4. [Serializers](#serializers)
5. [Views & ViewSets](#views--viewsets)
6. [Authentication & Permissions](#authentication--permissions)
7. [Admin Interface (Jazzmin)](#admin-interface-jazzmin)
8. [Signals & Events](#signals--events)
9. [Services Layer](#services-layer)
10. [Caching Strategies](#caching-strategies)
11. [Async & Channels](#async--channels)
12. [Performance Optimization](#performance-optimization)
13. [Docker Configuration](#docker-configuration)
14. [Testing Strategies](#testing-strategies)
15. [Monitoring & Observability](#monitoring--observability)

---

## Project Architecture

### Directory Structure
```
backend/
├── apps/
│   ├── core/           # Base models, mixins, utilities
│   ├── accounts/       # User management, authentication
│   ├── api/           # API versioning, common serializers
│   └── [domain_apps]/ # Business logic apps
├── config/            # Django settings
├── services/          # Business logic layer
├── adapters/         # External integrations
├── utils/            # Shared utilities
├── Dockerfile
├── requirements/     # Environment-specific requirements
└── manage.py
```

### Design Principles
- **Separation of Concerns**: Models for data, serializers for transformation, views for HTTP handling, services for business logic
- **Domain-Driven Design**: Organize apps by business domains, not technical layers
- **Dependency Injection**: Use dependency injection for external services and complex business logic
- **Fail Fast**: Validate inputs early, use type hints, embrace explicit error handling

---

## Development Environment

### Essential Packages
```python
# Core Framework
Django>=5.1
djangorestframework>=3.15
django-cors-headers
django-extensions

# Performance & Optimization
django-cachalot          # ORM caching
django-silk             # Profiling
django-compression      # Response compression
django-ratelimit       # Rate limiting

# Authentication & Security
djangorestframework-simplejwt
django-oauth-toolkit
django-guardian        # Object-level permissions
django-axes           # Brute force protection

# Database & Caching
psycopg2-binary       # PostgreSQL adapter
redis>=4.0           # Caching & sessions
celery>=5.3          # Background tasks
django-celery-beat   # Periodic tasks

# Admin Interface
django-jazzmin       # Modern admin UI

# Async & Real-time
channels>=4.0        # WebSocket support
channels-redis      # Channel layers

# Development & Testing
django-debug-toolbar
factory-boy         # Test factories
pytest-django      # Testing framework
pre-commit         # Code quality

# Documentation
drf-spectacular    # OpenAPI 3.0 schema
```

### Settings Architecture
Structure settings for different environments:

- `settings/base.py` - Common settings
- `settings/development.py` - Development overrides
- `settings/production.py` - Production optimizations
- `settings/testing.py` - Test-specific settings

### Environment Variables
Use `django-environ` for configuration management:
- Database credentials
- Redis connection strings
- Secret keys and API tokens
- Feature flags
- Performance tuning parameters

---

## Database & Models

### Model Design Best Practices

#### Base Model Pattern
Create abstract base models for common fields and behaviors:
```python
# apps/core/models.py
class TimestampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True

class SoftDeleteModel(models.Model):
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        abstract = True
```

#### Performance Considerations
- **Indexes**: Add `db_index=True` for frequently queried fields
- **Select Related**: Use `select_related` and `prefetch_related` managers
- **Constraints**: Use database constraints for data integrity
- **Field Types**: Choose appropriate field types (avoid `TextField` for short strings)

#### Model Managers
Create custom managers for common query patterns:
```python
class ActiveManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_active=True)

class Model(TimestampedModel):
    is_active = models.BooleanField(default=True)
    
    objects = models.Manager()  # Default manager
    active = ActiveManager()    # Custom manager
```

#### Validation
- Use `clean()` methods for complex validation
- Implement custom field validators
- Use `unique_together` and `unique` constraints
- Consider using `django-model-utils` for common patterns

---

## Serializers

### Serializer Architecture

#### Base Serializers
Create base serializers for common patterns:
```python
# apps/core/serializers.py
class BaseModelSerializer(serializers.ModelSerializer):
    """Base serializer with common optimizations"""
    
    class Meta:
        abstract = True
    
    def __init__(self, *args, **kwargs):
        # Dynamic field exclusion
        exclude_fields = kwargs.pop('exclude_fields', None)
        if exclude_fields:
            for field in exclude_fields:
                self.fields.pop(field, None)
        super().__init__(*args, **kwargs)
```

#### Performance Optimizations
- **Field Selection**: Use `fields` and `exclude` strategically
- **Depth Control**: Limit serializer depth to prevent N+1 queries
- **Method Fields**: Use `SerializerMethodField` sparingly
- **Validation Caching**: Cache expensive validation logic

#### Nested Serializers
Handle nested relationships efficiently:
- Use `PrimaryKeyRelatedField` for writes
- Use nested serializers only for reads
- Implement custom `to_representation()` for complex data

#### Validation Strategies
- Field-level validation with `validate_<field_name>`
- Object-level validation with `validate()`
- Cross-field validation in `validate()`
- Use `ValidationError` with proper error codes

---

## Views & ViewSets

### View Architecture

#### Base ViewSets
Create base viewsets with common functionality:
```python
# apps/core/views.py
class BaseModelViewSet(viewsets.ModelViewSet):
    """Base viewset with common optimizations"""
    
    # Performance optimizations
    prefetch_for_list = []
    prefetch_for_detail = []
    
    def get_queryset(self):
        queryset = super().get_queryset()
        
        if self.action == 'list':
            return queryset.prefetch_related(*self.prefetch_for_list)
        elif self.action == 'retrieve':
            return queryset.select_related(*self.prefetch_for_detail)
        
        return queryset
```

#### Async Views for I/O Heavy Operations
Use async views for external API calls and file operations:
```python
from django.http import JsonResponse
import asyncio
import aiohttp

async def async_api_view(request):
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.example.com/data') as response:
            data = await response.json()
    
    return JsonResponse({'data': data})
```

#### Response Optimization
- Use pagination for all list endpoints
- Implement efficient filtering with `django-filter`
- Use `select_related` and `prefetch_related` in querysets
- Cache expensive computations

#### Error Handling
Implement consistent error handling:
- Use custom exception handlers
- Return structured error responses
- Log errors with context
- Handle validation errors gracefully

---

## Authentication & Permissions

### JWT Authentication Strategy
Configure JWT with proper security settings:
- Short access token lifetime (15-30 minutes)
- Longer refresh token lifetime (7-30 days)
- Token rotation on refresh
- Blacklist revoked tokens

### Permission Classes
Create granular permission classes:
```python
# apps/core/permissions.py
class IsOwnerOrReadOnly(BasePermission):
    """Custom permission for object ownership"""
    
    def has_object_permission(self, request, view, obj):
        if request.method in SAFE_METHODS:
            return True
        return obj.owner == request.user
```

### Rate Limiting
Implement rate limiting at multiple levels:
- Global rate limits
- Per-user rate limits
- Per-endpoint rate limits
- Anonymous vs authenticated limits

### Security Headers
Configure security headers:
- CORS settings for cross-origin requests
- CSP headers for XSS protection
- HSTS for HTTPS enforcement
- Content type validation

---

## Admin Interface (Jazzmin)

### Jazzmin Configuration
Configure Jazzmin for optimal admin experience:

```python
# settings.py
JAZZMIN_SETTINGS = {
    "site_title": "Your App Admin",
    "site_header": "Your App",
    "site_brand": "Your App",
    "welcome_sign": "Welcome to Your App Admin",
    
    # UI Tweaks
    "show_sidebar": True,
    "navigation_expanded": True,
    
    # Icons
    "icons": {
        "auth": "fas fa-users-cog",
        "auth.user": "fas fa-user",
        "auth.Group": "fas fa-users",
    },
    
    # Custom CSS/JS
    "custom_css": "admin/custom.css",
    "custom_js": "admin/custom.js",
}
```

### Admin Optimizations
- Use `list_select_related` and `list_prefetch_related`
- Implement efficient search with `search_fields`
- Use `list_filter` for common filtering
- Override `get_queryset()` for performance

### Custom Admin Actions
Create bulk actions for common operations:
- Bulk status updates
- Export functionalities
- Batch processing actions

---

## Signals & Events

### Signal Best Practices
- Keep signal handlers lightweight
- Use `post_save` for side effects, not business logic
- Implement idempotent signal handlers
- Consider using `django-lifecycle` for model lifecycle management

### Event-Driven Architecture
Implement event sourcing patterns:
- Domain events for business logic
- Event handlers for side effects
- Async event processing with Celery
- Event store for audit trails

### Decoupling with Events
Use events to decouple applications:
- Publisher-subscriber patterns
- Event buses for cross-app communication
- Webhook systems for external integrations

---

## Services Layer

### Service Architecture
Create services for complex business logic:
```python
# services/user_service.py
class UserService:
    def __init__(self, repository=None, email_service=None):
        self.repository = repository or UserRepository()
        self.email_service = email_service or EmailService()
    
    def create_user(self, data):
        # Complex user creation logic
        user = self.repository.create(data)
        self.email_service.send_welcome_email(user)
        return user
```

### Repository Pattern
Abstract database access through repositories:
- Centralize query logic
- Enable easier testing with mocks
- Support multiple data sources

### Domain Services
Implement domain-specific services:
- Payment processing services
- Notification services
- File upload services
- Integration services

---

## Caching Strategies

### Multi-Level Caching
Implement caching at multiple levels:

#### 1. Database Query Caching
Use `django-cachalot` for automatic ORM caching:
```python
INSTALLED_APPS = ['cachalot']
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://redis:6379/1',
    }
}
```

#### 2. View-Level Caching
Cache expensive view computations:
```python
from django.core.cache import cache

def expensive_view(request):
    cache_key = f"expensive_view_{request.user.id}"
    result = cache.get(cache_key)
    
    if result is None:
        result = perform_expensive_computation()
        cache.set(cache_key, result, timeout=300)
    
    return Response(result)
```

#### 3. Response Caching
Use HTTP caching headers:
- ETags for conditional requests
- Cache-Control headers
- Vary headers for personalized content

#### 4. Cache Invalidation
Implement smart cache invalidation:
- Tag-based invalidation
- Time-based expiration
- Event-driven cache clearing

---

## Async & Channels

### WebSocket Implementation
Configure Django Channels for WebSocket support, extending Django's abilities beyond HTTP to handle WebSockets, MQTT, and other protocols while preserving Django's synchronous nature:

#### Channel Layers Configuration
```python
# settings.py
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            "hosts": [('redis', 6379)],
            "capacity": 1500,
            "expiry": 10,
        },
    },
}
```

#### Consumer Patterns
Create efficient WebSocket consumers:
- Use async consumers for I/O operations
- Implement proper error handling
- Use channel groups for broadcasting
- Handle connection lifecycle properly

#### MQTT Support
Integrate MQTT for IoT applications:
- Use `paho-mqtt` for MQTT client
- Bridge MQTT messages to WebSocket clients
- Implement message queuing and reliability

### Async View Strategies
- Use async views for external API calls
- Keep database operations in sync views
- Use `sync_to_async` and `async_to_sync` carefully
- Monitor async task performance

---

## Performance Optimization

### Database Optimization

#### Query Optimization
- Use `select_related()` for foreign keys
- Use `prefetch_related()` for many-to-many and reverse foreign keys
- Implement custom prefetches with `Prefetch()`
- Use `only()` and `defer()` for field selection

#### Indexing Strategy
```python
class Model(models.Model):
    # Single field index
    name = models.CharField(max_length=100, db_index=True)
    
    # Composite index
    class Meta:
        indexes = [
            models.Index(fields=['status', 'created_at']),
            models.Index(fields=['user', '-created_at']),
        ]
```

#### Connection Pooling
Configure database connection pooling:
```python
# Use django-db-pool for connection pooling
DATABASES = {
    'default': {
        'ENGINE': 'django_db_pool.backends.postgresql',
        'POOL_OPTIONS': {
            'POOL_SIZE': 10,
            'MAX_OVERFLOW': 10,
        }
    }
}
```

### Response Optimization
- Use `gzip` compression for responses
- Implement pagination for all list endpoints
- Use efficient serializer patterns
- Cache serialized responses

### Async Task Processing
Use Celery for background tasks:
- Email sending
- File processing
- Report generation
- Periodic cleanup tasks

---

## Docker Configuration

### Dockerfile
Create optimized Docker images:

```dockerfile
# Use Python slim image
FROM python:3.11-slim

# Set environment variables
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Create and set working directory
WORKDIR /app

# Install Python dependencies
COPY requirements/production.txt requirements/
RUN pip install --no-cache-dir -r requirements/production.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd --create-home --shell /bin/bash app
RUN chown -R app:app /app
USER app

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

# Command
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

### Multi-Stage Builds
Use multi-stage builds for smaller production images:
- Build stage for dependencies
- Production stage with only runtime requirements
- Static file collection in build stage

### Docker Optimizations
- Use `.dockerignore` to reduce context
- Layer caching for faster builds
- Security scanning with `docker scan`
- Image size optimization

---

## Testing Strategies

### Test Architecture
Structure tests for maintainability:
```
tests/
├── unit/           # Unit tests for individual components
├── integration/    # Integration tests
├── api/           # API endpoint tests
├── factories/     # Test data factories
└── fixtures/      # Test fixtures
```

### Test Data Management
Use Factory Boy for test data:
- Create realistic test data
- Use traits for variations
- Implement data relationships
- Use subfactories for complex objects

### API Testing
Test API endpoints thoroughly:
- Authentication and permissions
- Input validation
- Response formatting
- Error handling
- Performance under load

### Performance Testing
- Use `django-silk` for profiling
- Load testing with `locust`
- Database query analysis
- Memory usage monitoring

---

## Monitoring & Observability

### Application Metrics
Track key application metrics:
- Request rate and response times
- Error rates by endpoint
- Database query performance
- Cache hit rates
- Background task processing

### Health Checks
Implement comprehensive health checks:
- Database connectivity
- Redis availability
- External service dependencies
- System resource usage

### Logging Strategy
Structure logging for production:
- Use structured logging (JSON format)
- Include correlation IDs
- Log security events
- Implement log aggregation

### Error Tracking
Integrate error tracking:
- Use Sentry for error monitoring
- Track performance issues
- Monitor security incidents
- Set up alerting thresholds

---

## Scaling Considerations

### Horizontal Scaling
Design for horizontal scaling:
- Stateless application servers
- Shared session storage (Redis)
- Load balancer configuration
- Database read replicas

### Caching Architecture
Implement distributed caching:
- Redis cluster for high availability
- Cache warming strategies
- Cache invalidation patterns
- Regional cache distribution

### Queue Management
Scale background processing:
- Multiple Celery workers
- Queue partitioning by priority
- Dead letter queues
- Monitoring and alerting

### Database Scaling
Plan database scaling:
- Read replicas for read-heavy workloads
- Connection pooling (PgBouncer)
- Query optimization
- Partitioning strategies

---

## Security Best Practices

### Input Validation
- Validate all inputs at multiple layers
- Use serializer validation
- Implement rate limiting
- Sanitize user-generated content

### Authentication Security
- Use strong password policies
- Implement MFA where possible
- Token rotation and expiration
- Secure session management

### Data Protection
- Encrypt sensitive data at rest
- Use HTTPS for all communications
- Implement proper access controls
- Regular security audits

### Dependency Management
- Keep dependencies updated
- Use security scanners
- Monitor CVE databases
- Implement dependency pinning

---

## Production Checklist

### Pre-Deployment
- [ ] Environment-specific settings configured
- [ ] Database migrations tested
- [ ] Static files properly collected
- [ ] Security settings reviewed
- [ ] Performance benchmarks met

### Infrastructure
- [ ] Load balancer configured
- [ ] SSL certificates installed
- [ ] Monitoring systems active
- [ ] Backup procedures tested
- [ ] Scaling policies defined

### Operational
- [ ] Log aggregation working
- [ ] Error tracking configured
- [ ] Health checks implemented
- [ ] Documentation updated
- [ ] Runbooks prepared

---

## Common Django App Use Cases & Patterns

### 1. User Management & Authentication Apps

#### Core Components
- **Models**: Custom User model, UserProfile, roles/groups
- **Serializers**: Registration, login, profile update, password change
- **Views**: Authentication endpoints, profile management, user listings
- **Services**: Email verification, password reset, social auth integration

#### Implementation Patterns
**Registration Flow**:
- Email verification with token-based confirmation
- Welcome email with onboarding links
- Default permissions and group assignment
- Profile creation with sensible defaults

**Profile Management**:
- Avatar upload with image optimization
- Progressive profile completion tracking
- Privacy settings and visibility controls
- Account deactivation/deletion workflows

**Permission Systems**:
- Role-based access control (RBAC)
- Object-level permissions for user-owned resources
- Team/organization membership management
- Feature flags and subscription-based access

---

### 2. Content Management Apps (Blog, CMS, Articles)

#### Core Components
- **Models**: Category, Tag, Article/Post, Comment, Media
- **Serializers**: Content creation/editing, metadata handling, relationships
- **Views**: Public content API, admin content management, search/filtering
- **Services**: Content publishing workflows, SEO optimization, media processing

#### Implementation Patterns
**Content Lifecycle**:
- Draft → Review → Published → Archived states
- Version control and revision history
- Scheduled publishing with Celery tasks
- Content approval workflows

**Rich Content Handling**:
- WYSIWYG editor integration
- Image/video upload and optimization
- SEO metadata management (title, description, keywords)
- Social media preview generation

**Engagement Features**:
- Comment system with moderation
- Like/reaction systems
- Content sharing and bookmarking
- Related content recommendations

---

### 3. E-commerce & Transaction Apps

#### Core Components
- **Models**: Product, Category, Cart, Order, Payment, Inventory
- **Serializers**: Product catalog, cart management, checkout process
- **Views**: Product listings, cart operations, order management
- **Services**: Payment processing, inventory management, order fulfillment

#### Implementation Patterns
**Product Management**:
- Variant handling (size, color, etc.)
- Inventory tracking and low-stock alerts
- Pricing rules and discount systems
- Product recommendation algorithms

**Shopping Cart**:
- Session-based carts for anonymous users
- Persistent carts for authenticated users
- Cart abandonment recovery
- Promo code and coupon systems

**Order Processing**:
- Multi-step checkout process
- Payment gateway integration (Stripe, PayPal)
- Order status tracking and notifications
- Return and refund management

---

### 4. Notification & Communication Apps

#### Core Components
- **Models**: Notification, NotificationPreference, Message, Thread
- **Serializers**: Notification formatting, message composition
- **Views**: Notification management, messaging endpoints
- **Services**: Multi-channel delivery, real-time updates

#### Implementation Patterns
**Notification System**:
- In-app, email, SMS, push notification channels
- User preference management per notification type
- Batch processing for bulk notifications
- Notification templates and localization

**Real-time Messaging**:
- WebSocket integration for instant messaging
- Message threading and conversation management
- File attachment and media sharing
- Message search and history

**Email Campaigns**:
- Template-based email system
- Subscriber management and segmentation
- A/B testing for email content
- Bounce handling and unsubscribe management

---

### 5. File Management & Media Apps

#### Core Components
- **Models**: File, Folder, Permission, DownloadLog
- **Serializers**: File upload/download, metadata handling
- **Views**: File operations, gallery views, permission management
- **Services**: Storage backend integration, media processing

#### Implementation Patterns
**File Upload System**:
- Chunked uploads for large files
- Multiple storage backends (local, S3, Google Cloud)
- Virus scanning and file validation
- Automatic thumbnail/preview generation

**Access Control**:
- Fine-grained file permissions
- Temporary download links with expiration
- Bandwidth throttling for downloads
- Audit logging for file access

**Media Processing**:
- Image resizing and optimization
- Video transcoding for multiple formats
- Document preview generation (PDF, Office docs)
- Metadata extraction (EXIF, document properties)

---

### 6. Analytics & Reporting Apps

#### Core Components
- **Models**: Event, Metric, Report, Dashboard
- **Serializers**: Analytics data formatting, chart configurations
- **Views**: Data aggregation endpoints, report generation
- **Services**: Real-time analytics, data export

#### Implementation Patterns
**Event Tracking**:
- User behavior tracking
- Custom event definitions
- Real-time event processing
- Privacy-compliant data collection

**Report Generation**:
- Automated report scheduling
- Export to multiple formats (PDF, Excel, CSV)
- Interactive dashboard widgets
- Data visualization with Chart.js/D3.js

**Performance Monitoring**:
- Application performance metrics
- Error tracking and alerting
- User session recording
- A/B testing result analysis

---

### 7. Search & Discovery Apps

#### Core Components
- **Models**: SearchIndex, SearchQuery, Suggestion, Filter
- **Serializers**: Search results formatting, faceted search data
- **Views**: Search endpoints, autocomplete, advanced filtering
- **Services**: Search indexing, relevance scoring

#### Implementation Patterns
**Search Implementation**:
- Elasticsearch/Solr integration
- Full-text search with ranking
- Faceted search and filtering
- Search analytics and optimization

**Autocomplete & Suggestions**:
- Type-ahead search suggestions
- Popular search terms tracking
- Personalized search results
- Search result caching

**Content Discovery**:
- Recommendation engines
- Trending content algorithms
- Similar item suggestions
- User interest profiling

---

## Essential Implementation Details

### Pagination Strategies

#### 1. Offset-Based Pagination (DRF Default)
**Best For**: Small to medium datasets, simple navigation
**Implementation**: Use `PageNumberPagination` or `LimitOffsetPagination`
**Configuration**:
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    'PAGE_SIZE_QUERY_PARAM': 'page_size',
    'MAX_PAGE_SIZE': 100,
}
```

#### 2. Cursor-Based Pagination
**Best For**: Large datasets, real-time feeds, performance-critical applications
**Benefits**: Consistent performance, handles new items gracefully
**Use Cases**: Activity feeds, chat messages, time-series data

#### 3. Custom Pagination
**Best For**: Complex business requirements, optimized user experience
**Features**: Different pagination methods with Django and Postgres, explaining benefits and tradeoffs
- Load more buttons
- Infinite scrolling
- Hybrid approaches

### Filtering & Search Patterns

#### QuerySet Optimization
- Use `select_related()` for foreign key lookups
- Use `prefetch_related()` for reverse foreign keys and many-to-many
- Implement `only()` and `defer()` for field selection
- Add database indexes for frequently filtered fields

#### Advanced Filtering
```python
# Use django-filter for complex filtering
class ProductFilter(django_filters.FilterSet):
    price_range = django_filters.RangeFilter(field_name='price')
    category = django_filters.ModelChoiceFilter(queryset=Category.objects.all())
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='gte')
    
    class Meta:
        model = Product
        fields = ['name', 'description', 'is_active']
```

#### Search Implementation
- Use PostgreSQL full-text search for basic needs
- Implement Elasticsearch for advanced search requirements
- Add search result ranking and relevance scoring
- Track search queries for analytics

### Caching Strategies by Use Case

#### View-Level Caching
- Cache expensive database queries
- Use cache keys with user context
- Implement cache invalidation strategies
- Monitor cache hit rates

#### Object-Level Caching
```python
# Cache individual model instances
def get_cached_product(product_id):
    cache_key = f'product_{product_id}'
    product = cache.get(cache_key)
    
    if product is None:
        product = Product.objects.select_related('category').get(id=product_id)
        cache.set(cache_key, product, timeout=3600)  # 1 hour
    
    return product
```

#### Query Result Caching
- Cache expensive aggregation queries
- Use `django-cachalot` for automatic ORM caching
- Implement smart cache warming
- Handle cache stampede scenarios

---

## Django App Development Checklist

### 📋 Individual Django App Checklist

#### Models & Database
- [ ] **Model Design**
  - [ ] Models follow single responsibility principle
  - [ ] Proper field types chosen (avoid TextField for short strings)
  - [ ] Foreign keys and relationships properly defined
  - [ ] Custom managers created for common query patterns
  - [ ] Abstract base models used for common fields (timestamps, soft delete)
  
- [ ] **Database Optimization**
  - [ ] Database indexes added for frequently queried fields
  - [ ] Unique constraints and database-level validations implemented
  - [ ] Composite indexes for multi-field queries
  - [ ] Migration files are clean and reversible
  
- [ ] **Model Validation**
  - [ ] Custom validation methods implemented (`clean()`, field validators)
  - [ ] Business logic constraints enforced at model level
  - [ ] String representations (`__str__`) are meaningful
  - [ ] Model meta options configured (ordering, permissions, etc.)

#### Serializers
- [ ] **Serializer Structure**
  - [ ] Appropriate serializer types chosen (Model, Regular, Nested)
  - [ ] Field selection optimized (`fields`, `exclude` used strategically)
  - [ ] Nested serializers handled efficiently
  - [ ] Write vs. read serializers separated when needed
  
- [ ] **Validation & Security**
  - [ ] All user inputs validated at serializer level
  - [ ] Cross-field validation implemented where needed
  - [ ] Sensitive fields excluded from responses
  - [ ] Custom validation methods documented
  
- [ ] **Performance**
  - [ ] SerializerMethodField usage minimized
  - [ ] Expensive computations cached or moved to services
  - [ ] Dynamic field exclusion implemented where beneficial
  - [ ] Serializer depth limited to prevent N+1 queries

#### Views & ViewSets
- [ ] **View Architecture**
  - [ ] Appropriate view types chosen (APIView, ViewSet, GenericView)
  - [ ] QuerySet optimization implemented (select_related, prefetch_related)
  - [ ] Custom actions documented and secured
  - [ ] HTTP methods properly mapped to actions
  
- [ ] **Performance & Pagination**
  - [ ] **Pagination implemented on all list endpoints**
  - [ ] **Page size limits configured**
  - [ ] **Pagination style chosen based on use case**
  - [ ] QuerySet filtering optimized
  - [ ] Expensive operations moved to background tasks
  
- [ ] **Error Handling**
  - [ ] Custom exception handling implemented
  - [ ] Proper HTTP status codes returned
  - [ ] Error responses are consistent and informative
  - [ ] Validation errors properly formatted

#### Authentication & Permissions
- [ ] **Access Control**
  - [ ] Permission classes assigned to all views
  - [ ] Object-level permissions implemented where needed
  - [ ] Authentication requirements clearly defined
  - [ ] Rate limiting configured for public endpoints
  
- [ ] **Security**
  - [ ] Input validation against injection attacks
  - [ ] File upload security (if applicable)
  - [ ] CORS settings configured appropriately
  - [ ] Sensitive data excluded from logs and responses

#### Services & Business Logic
- [ ] **Service Layer**
  - [ ] Complex business logic moved to service classes
  - [ ] Services are testable and loosely coupled
  - [ ] External API integrations properly handled
  - [ ] Transaction boundaries clearly defined
  
- [ ] **Background Tasks**
  - [ ] Long-running operations moved to Celery tasks
  - [ ] Task failure handling and retries configured
  - [ ] Task monitoring and alerting set up
  - [ ] Idempotent task design for reliability

#### Admin Interface
- [ ] **Admin Configuration**
  - [ ] All models registered in admin
  - [ ] List display fields optimized
  - [ ] Search and filter options configured
  - [ ] Custom admin actions implemented where needed
  - [ ] Admin performance optimized (list_select_related, etc.)

#### Testing
- [ ] **Test Coverage**
  - [ ] Unit tests for models, serializers, services
  - [ ] API endpoint tests for all views
  - [ ] Permission and authentication tests
  - [ ] Edge cases and error scenarios tested
  - [ ] Test factories created for data generation
  
- [ ] **Test Quality**
  - [ ] Tests are isolated and independent
  - [ ] Test database performance acceptable
  - [ ] Mock external dependencies properly
  - [ ] Integration tests for critical workflows

#### Documentation
- [ ] **API Documentation**
  - [ ] API endpoints documented with drf-spectacular
  - [ ] Request/response examples provided
  - [ ] Authentication requirements documented
  - [ ] Rate limits and constraints documented
  
- [ ] **Code Documentation**
  - [ ] Complex business logic documented
  - [ ] Service interfaces clearly defined
  - [ ] Model relationships explained
  - [ ] Configuration options documented

---

### 🏗️ Entire Django Project Checklist

#### Project Structure & Configuration
- [ ] **Architecture**
  - [ ] Apps organized by business domain, not technical layer
  - [ ] Settings split by environment (dev, staging, prod)
  - [ ] Environment variables used for configuration
  - [ ] Requirements files organized by environment
  - [ ] Proper Python package structure maintained
  
- [ ] **Dependencies**
  - [ ] Dependencies pinned to specific versions
  - [ ] Security vulnerabilities checked regularly
  - [ ] Unused dependencies removed
  - [ ] Requirements files kept up to date

#### Database & Performance
- [ ] **Database Configuration**
  - [ ] Connection pooling configured (PgBouncer)
  - [ ] Database backups automated and tested
  - [ ] Migration strategy documented
  - [ ] Database performance monitoring set up
  
- [ ] **Caching Strategy**
  - [ ] Redis configured for caching and sessions
  - [ ] Cache invalidation strategies implemented
  - [ ] Cache performance monitored
  - [ ] Different cache backends for different use cases
  
- [ ] **Query Optimization**
  - [ ] N+1 query problems eliminated
  - [ ] Database query performance profiled
  - [ ] Slow query logging enabled
  - [ ] Database indexes optimized

#### Security & Authentication
- [ ] **Security Configuration**
  - [ ] Security headers configured (HSTS, CSP, etc.)
  - [ ] HTTPS enforced in production
  - [ ] Debug mode disabled in production
  - [ ] Secret keys properly managed
  - [ ] SQL injection and XSS protection verified
  
- [ ] **Authentication System**
  - [ ] JWT configuration secure (short access tokens)
  - [ ] Password policies enforced
  - [ ] Rate limiting implemented
  - [ ] Session security configured
  - [ ] Multi-factor authentication considered

#### API Design & Standards
- [ ] **RESTful Design**
  - [ ] Consistent URL patterns across apps
  - [ ] Proper HTTP methods and status codes
  - [ ] API versioning strategy implemented
  - [ ] Resource naming conventions followed
  
- [ ] **Response Standards**
  - [ ] Consistent error response format
  - [ ] Pagination implemented on all list endpoints
  - [ ] Response compression enabled
  - [ ] CORS configured appropriately
  
- [ ] **API Documentation**
  - [ ] Complete API documentation available
  - [ ] Interactive API explorer configured
  - [ ] Authentication examples provided
  - [ ] Postman/Insomnia collections available

#### Async & Real-time Features
- [ ] **Channels Configuration**
  - [ ] WebSocket support configured if needed
  - [ ] Channel layers properly configured with Redis
  - [ ] Consumer classes properly implemented
  - [ ] WebSocket authentication handled
  
- [ ] **Background Processing**
  - [ ] Celery workers configured and monitored
  - [ ] Task queues organized by priority
  - [ ] Task failure handling implemented
  - [ ] Periodic tasks scheduled appropriately

#### Deployment & Infrastructure
- [ ] **Docker Configuration**
  - [ ] Dockerfile optimized for production
  - [ ] Multi-stage builds implemented
  - [ ] Health checks configured
  - [ ] Security scanning enabled
  
- [ ] **Environment Management**
  - [ ] Separate environments for dev/staging/prod
  - [ ] Environment-specific configurations
  - [ ] Secrets management implemented
  - [ ] Feature flags system considered

#### Monitoring & Observability
- [ ] **Application Monitoring**
  - [ ] Error tracking configured (Sentry)
  - [ ] Application metrics collected
  - [ ] Performance monitoring set up
  - [ ] Health check endpoints implemented
  
- [ ] **Logging**
  - [ ] Structured logging implemented
  - [ ] Log levels configured appropriately
  - [ ] Sensitive data excluded from logs
  - [ ] Log aggregation configured
  
- [ ] **Alerting**
  - [ ] Critical error alerts configured
  - [ ] Performance threshold alerts set up
  - [ ] Infrastructure monitoring integrated
  - [ ] On-call procedures documented

#### Testing & Quality Assurance
- [ ] **Test Coverage**
  - [ ] Overall test coverage above 80%
  - [ ] Critical paths have 100% coverage
  - [ ] Integration tests for key workflows
  - [ ] Performance tests for critical endpoints
  
- [ ] **Code Quality**
  - [ ] Pre-commit hooks configured
  - [ ] Code formatting standards enforced
  - [ ] Static analysis tools integrated
  - [ ] Continuous integration set up

#### Documentation & Knowledge Management
- [ ] **Project Documentation**
  - [ ] README with setup instructions
  - [ ] API documentation complete
  - [ ] Deployment procedures documented
  - [ ] Architecture decisions recorded
  
- [ ] **Team Knowledge**
  - [ ] Code review processes established
  - [ ] Development standards documented
  - [ ] Troubleshooting guides created
  - [ ] Onboarding documentation available

#### Scalability & Performance
- [ ] **Horizontal Scaling**
  - [ ] Application servers are stateless
  - [ ] Session storage externalized
  - [ ] Load balancing configured
  - [ ] Auto-scaling policies defined
  
- [ ] **Performance Benchmarks**
  - [ ] Load testing completed for target traffic
  - [ ] Performance benchmarks established
  - [ ] Bottlenecks identified and addressed
  - [ ] Scaling plans documented

#### Data Management & Compliance
- [ ] **Data Protection**
  - [ ] Personal data handling compliant (GDPR, etc.)
  - [ ] Data retention policies implemented
  - [ ] Data anonymization procedures
  - [ ] Backup and recovery tested
  
- [ ] **Audit & Compliance**
  - [ ] Audit logging implemented for security tracking
  - [ ] User activity logging configured
  - [ ] Compliance requirements met
  - [ ] Security audit procedures established

---

## Quick Reference Checklists

### 🚀 Pre-Deployment Checklist
- [ ] All environment variables configured
- [ ] Database migrations applied and tested
- [ ] Static files collection working
- [ ] Health checks responding
- [ ] Error tracking configured
- [ ] Monitoring dashboards set up
- [ ] Backup procedures tested
- [ ] SSL certificates configured
- [ ] Load balancer health checks working
- [ ] Performance benchmarks met

### 🔍 Code Review Checklist
- [ ] Security vulnerabilities checked
- [ ] Performance impact assessed
- [ ] Test coverage adequate
- [ ] Documentation updated
- [ ] Error handling implemented
- [ ] Logging added for debugging
- [ ] Database queries optimized
- [ ] Code follows project standards

### 🐛 Troubleshooting Checklist
- [ ] Check application logs for errors
- [ ] Verify database connectivity
- [ ] Check Redis/cache connectivity
- [ ] Review recent deployments
- [ ] Check system resource usage
- [ ] Verify external service status
- [ ] Review monitoring dashboards
- [ ] Check security logs for anomalies

---

## Recommended Third-Party Resources

### Documentation & Guides
- [Django REST Framework Documentation](https://www.django-rest-framework.org/)
- [Django Channels Documentation](https://channels.readthedocs.io/)
- [Celery Documentation](https://docs.celeryproject.org/)
- [Redis Documentation](https://redis.io/documentation)

### Performance Resources
- [Django Performance Optimization](https://docs.djangoproject.com/en/stable/topics/performance/)
- [Database Performance Tuning](https://www.postgresql.org/docs/current/performance-tips.html)
- [Python Profiling Tools](https://docs.python.org/3/library/profile.html)

### Security Resources
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Django Security Documentation](https://docs.djangoproject.com/en/stable/topics/security/)
- [Security Headers](https://securityheaders.com/)

---

This guide provides a foundation for building production-ready Django DRF applications that can scale to handle thousands of concurrent requests while maintaining clean, maintainable code. Remember to adapt these practices to your specific use case and always benchmark your optimizations.