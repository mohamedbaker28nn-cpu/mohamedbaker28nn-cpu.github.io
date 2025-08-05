# Technical Implementation Specifications

## API Design Specifications

### REST API Endpoints

#### Tenant Management API
```yaml
/api/tenants:
  POST:
    summary: Create new tenant
    body:
      email: string
      subdomain: string
      brandingConfig: object
    responses:
      201: Tenant created successfully
      409: Subdomain already exists

  GET /{tenantId}:
    summary: Get tenant details
    responses:
      200: Tenant data
      404: Tenant not found

  PATCH /{tenantId}:
    summary: Update tenant settings
    body:
      brandingConfig: object
      customDomain: string
    responses:
      200: Updated successfully
      403: Unauthorized

/api/tenants/{tenantId}/courses:
  GET:
    summary: List tenant courses
    query:
      page: number
      limit: number
      published: boolean
    responses:
      200: Paginated course list

  POST:
    summary: Create new course
    body:
      title: string
      description: string
      thumbnail: file
    responses:
      201: Course created
      400: Validation error
```

#### Course Management API
```yaml
/api/courses/{courseId}:
  GET:
    summary: Get course details
    responses:
      200: Course data with lectures
      404: Course not found

  PATCH:
    summary: Update course
    body:
      title: string
      description: string
      published: boolean
    responses:
      200: Updated successfully

  DELETE:
    summary: Delete course
    responses:
      204: Deleted successfully
      409: Course has enrolled students

/api/courses/{courseId}/lectures:
  POST:
    summary: Create lecture
    body:
      title: string
      videoFile: file
      order: number
    responses:
      201: Lecture created, processing started
      413: File too large

/api/courses/{courseId}/access-codes:
  POST:
    summary: Generate access codes
    body:
      quantity: number
      expiresAt: date
    responses:
      201: Codes generated
      400: Invalid parameters

  GET:
    summary: List access codes
    responses:
      200: Code list with usage status
```

#### Student API
```yaml
/api/students/enroll:
  POST:
    summary: Enroll with access code
    body:
      code: string
      studentInfo: object
    responses:
      200: Enrollment successful
      404: Invalid code
      410: Code already used

/api/students/courses:
  GET:
    summary: Get enrolled courses
    responses:
      200: Course list with progress

/api/students/courses/{courseId}/lectures/{lectureId}:
  GET:
    summary: Get lecture video URL
    responses:
      200: Signed video URL
      403: Not enrolled
```

### WebSocket Events

#### Real-time Video Processing Status
```typescript
// Client subscribes to processing updates
interface VideoProcessingEvent {
  lectureId: string;
  status: 'queued' | 'processing' | 'completed' | 'failed';
  progress?: number;
  error?: string;
}

// Server emits events
socket.emit('video:processing', {
  lectureId: 'uuid',
  status: 'processing',
  progress: 45
});
```

#### Live Dashboard Updates
```typescript
// Dashboard analytics updates
interface DashboardEvent {
  tenantId: string;
  metric: 'new_enrollment' | 'course_view' | 'video_watch';
  data: any;
}
```

## Database Optimization

### Query Optimization Strategies

#### Tenant-Scoped Queries
```sql
-- Efficient tenant filtering with proper indexing
SELECT c.*, COUNT(e.id) as student_count
FROM courses c
LEFT JOIN enrollments e ON c.id = e.course_id
WHERE c.tenant_id = $1
  AND c.published = true
GROUP BY c.id
ORDER BY c.created_at DESC
LIMIT 20 OFFSET $2;

-- Index to support this query
CREATE INDEX CONCURRENTLY idx_courses_tenant_published 
ON courses(tenant_id, published, created_at DESC);
```

#### Video Analytics Queries
```sql
-- Optimized video watch analytics
SELECT 
  l.title,
  AVG(wp.completion_percentage) as avg_completion,
  COUNT(DISTINCT wp.student_id) as unique_viewers
FROM lectures l
JOIN watch_progress wp ON l.id = wp.lecture_id
JOIN courses c ON l.course_id = c.id
WHERE c.tenant_id = $1
  AND wp.created_at >= NOW() - INTERVAL '30 days'
GROUP BY l.id, l.title
ORDER BY avg_completion DESC;

-- Supporting index
CREATE INDEX CONCURRENTLY idx_watch_progress_analytics
ON watch_progress(lecture_id, student_id, created_at, completion_percentage);
```

### Database Partitioning Strategy

#### Time-based Partitioning for Analytics
```sql
-- Partition watch_progress by month
CREATE TABLE watch_progress (
    id UUID DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL,
    lecture_id UUID NOT NULL,
    completion_percentage INTEGER DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE watch_progress_2024_01 PARTITION OF watch_progress
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE watch_progress_2024_02 PARTITION OF watch_progress
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

## Caching Strategy

### Redis Cache Patterns

#### Tenant Data Caching
```typescript
// Tenant cache with 1-hour TTL
export class TenantCacheService {
  async getTenant(domain: string): Promise<Tenant | null> {
    const cacheKey = `tenant:${domain}`;
    
    // Try cache first
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // Fetch from database
    const tenant = await this.tenantRepository.findByDomain(domain);
    if (tenant) {
      await this.redis.setex(cacheKey, 3600, JSON.stringify(tenant));
    }
    
    return tenant;
  }
  
  async invalidateTenant(domain: string): Promise<void> {
    await this.redis.del(`tenant:${domain}`);
  }
}
```

#### Course Data Caching
```typescript
// Course data with cache-aside pattern
export class CourseCacheService {
  async getCourseWithLectures(courseId: string): Promise<CourseWithLectures> {
    const cacheKey = `course:${courseId}:full`;
    
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    const course = await this.courseRepository.findWithLectures(courseId);
    if (course) {
      // Cache for 30 minutes
      await this.redis.setex(cacheKey, 1800, JSON.stringify(course));
    }
    
    return course;
  }
}
```

### Cloudflare Caching Rules

#### Landing Page Caching
```yaml
# Cloudflare Page Rules
"*.courseplatform.com/*":
  cache_level: "cache_everything"
  edge_cache_ttl: 86400  # 24 hours
  browser_cache_ttl: 3600  # 1 hour
  
"*/api/*":
  cache_level: "bypass"
  
"*/dashboard/*":
  cache_level: "bypass"
```

#### Video Stream Caching
```yaml
# R2 Video Content
"*/videos/*":
  cache_level: "cache_everything"
  edge_cache_ttl: 2592000  # 30 days
  browser_cache_ttl: 86400  # 24 hours
```

## Security Implementation

### Authentication Middleware

#### Tenant Context Middleware
```typescript
@Injectable()
export class TenantContextMiddleware implements NestMiddleware {
  constructor(
    private tenantService: TenantService,
    private cacheService: TenantCacheService
  ) {}
  
  async use(req: Request, res: Response, next: NextFunction) {
    const host = req.get('host');
    const subdomain = this.extractSubdomain(host);
    
    if (!subdomain) {
      return res.status(404).json({ error: 'Invalid domain' });
    }
    
    try {
      const tenant = await this.cacheService.getTenant(subdomain);
      if (!tenant) {
        return res.status(404).json({ error: 'Tenant not found' });
      }
      
      // Inject tenant context
      req['tenant'] = tenant;
      next();
    } catch (error) {
      return res.status(500).json({ error: 'Internal server error' });
    }
  }
  
  private extractSubdomain(host: string): string | null {
    const parts = host.split('.');
    return parts.length > 2 ? parts[0] : null;
  }
}
```

#### Role-based Authorization Guard
```typescript
@Injectable()
export class RoleGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  
  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(
      'roles',
      [context.getHandler(), context.getClass()]
    );
    
    if (!requiredRoles) {
      return true;
    }
    
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const tenant = request.tenant;
    
    // Check if user belongs to tenant
    if (user.tenantId !== tenant.id) {
      return false;
    }
    
    // Check role permissions
    return requiredRoles.some(role => user.roles?.includes(role));
  }
}
```

### Video Access Security

#### Signed URL Generation
```typescript
@Injectable()
export class VideoSecurityService {
  constructor(private configService: ConfigService) {}
  
  generateSignedUrl(videoPath: string, studentId: string, expiresIn: number = 3600): string {
    const secret = this.configService.get('VIDEO_SIGNING_SECRET');
    const expires = Math.floor(Date.now() / 1000) + expiresIn;
    
    const payload = {
      path: videoPath,
      student: studentId,
      exp: expires
    };
    
    const signature = crypto
      .createHmac('sha256', secret)
      .update(JSON.stringify(payload))
      .digest('hex');
    
    return `${videoPath}?signature=${signature}&expires=${expires}&student=${studentId}`;
  }
  
  validateSignedUrl(url: string, studentId: string): boolean {
    // Parse URL and validate signature
    const urlObj = new URL(url);
    const signature = urlObj.searchParams.get('signature');
    const expires = parseInt(urlObj.searchParams.get('expires'));
    const student = urlObj.searchParams.get('student');
    
    // Check expiration
    if (Date.now() / 1000 > expires) {
      return false;
    }
    
    // Verify student
    if (student !== studentId) {
      return false;
    }
    
    // Verify signature
    const secret = this.configService.get('VIDEO_SIGNING_SECRET');
    const payload = {
      path: urlObj.pathname,
      student: studentId,
      exp: expires
    };
    
    const expectedSignature = crypto
      .createHmac('sha256', secret)
      .update(JSON.stringify(payload))
      .digest('hex');
    
    return signature === expectedSignature;
  }
}
```

## Video Processing Implementation

### FFmpeg Processing Configuration

#### HLS Encoding Lambda Function
```typescript
export const videoProcessingHandler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    const message = JSON.parse(record.body);
    const { lectureId, s3Key, tenantId } = message;
    
    try {
      await processVideo(lectureId, s3Key, tenantId);
    } catch (error) {
      console.error('Video processing failed:', error);
      await updateLectureStatus(lectureId, 'failed', error.message);
    }
  }
};

async function processVideo(lectureId: string, s3Key: string, tenantId: string) {
  // Download video from S3
  const inputPath = `/tmp/${lectureId}-input.mp4`;
  await downloadFromS3(s3Key, inputPath);
  
  // Create output directory
  const outputDir = `/tmp/${lectureId}-output`;
  await fs.mkdir(outputDir, { recursive: true });
  
  // FFmpeg command for HLS encoding
  const ffmpegCommand = [
    '-i', inputPath,
    '-c:v', 'libx264',
    '-c:a', 'aac',
    '-preset', 'fast',
    '-crf', '23',
    '-sc_threshold', '0',
    '-g', '48',
    '-keyint_min', '48',
    '-hls_time', '6',
    '-hls_playlist_type', 'vod',
    '-b:v:0', '800k', '-s:v:0', '640x360',
    '-b:v:1', '1400k', '-s:v:1', '854x480',
    '-b:v:2', '2800k', '-s:v:2', '1280x720',
    '-b:v:3', '5000k', '-s:v:3', '1920x1080',
    '-var_stream_map', 'v:0,a:0 v:1,a:1 v:2,a:2 v:3,a:3',
    '-master_pl_name', 'master.m3u8',
    '-f', 'hls',
    `${outputDir}/stream_%v.m3u8`
  ];
  
  await execFFmpeg(ffmpegCommand);
  
  // Upload processed files to R2
  const videoUrl = await uploadToR2(outputDir, tenantId, lectureId);
  
  // Update database
  await updateLectureStatus(lectureId, 'completed', null, videoUrl);
  
  // Cleanup
  await fs.rm(inputPath);
  await fs.rm(outputDir, { recursive: true });
}
```

#### Processing Queue Management
```typescript
@Injectable()
export class VideoQueueService {
  constructor(
    @InjectQueue('video-processing') private videoQueue: Queue
  ) {}
  
  async queueVideoProcessing(lectureId: string, s3Key: string, tenantId: string) {
    const job = await this.videoQueue.add(
      'process-video',
      {
        lectureId,
        s3Key,
        tenantId,
        timestamp: Date.now()
      },
      {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 5000
        },
        removeOnComplete: 10,
        removeOnFail: 5
      }
    );
    
    return job.id;
  }
  
  async getProcessingStatus(jobId: string) {
    const job = await this.videoQueue.getJob(jobId);
    return {
      status: await job.getState(),
      progress: job.progress(),
      processedOn: job.processedOn,
      failedReason: job.failedReason
    };
  }
}
```

## Custom Domain Implementation

### Automated Domain Configuration

#### Namecheap Integration
```typescript
@Injectable()
export class NamecheapDomainService {
  constructor(private configService: ConfigService) {}
  
  async configureDomain(domain: string, tenantId: string): Promise<void> {
    const apiKey = this.configService.get('NAMECHEAP_API_KEY');
    const username = this.configService.get('NAMECHEAP_USERNAME');
    
    try {
      // Create DNS records
      await this.createDNSRecords(domain, tenantId);
      
      // Request SSL certificate
      await this.requestSSLCertificate(domain);
      
      // Update tenant record
      await this.updateTenantDomain(tenantId, domain);
      
    } catch (error) {
      throw new Error(`Domain configuration failed: ${error.message}`);
    }
  }
  
  private async createDNSRecords(domain: string, tenantId: string): Promise<void> {
    const platformDomain = this.configService.get('PLATFORM_DOMAIN');
    
    const records = [
      {
        type: 'CNAME',
        name: '@',
        value: platformDomain,
        ttl: 300
      },
      {
        type: 'CNAME',
        name: 'www',
        value: platformDomain,
        ttl: 300
      }
    ];
    
    for (const record of records) {
      await this.namecheapAPI.createRecord(domain, record);
    }
  }
  
  private async requestSSLCertificate(domain: string): Promise<void> {
    // Use AWS ACM for SSL certificate
    const acm = new AWS.ACM({ region: 'us-east-1' });
    
    const params = {
      DomainName: domain,
      SubjectAlternativeNames: [`www.${domain}`],
      ValidationMethod: 'DNS'
    };
    
    const result = await acm.requestCertificate(params).promise();
    
    // Store certificate ARN for later use
    await this.storeCertificateArn(domain, result.CertificateArn);
  }
}
```

#### DNS Validation Service
```typescript
@Injectable()
export class DNSValidationService {
  async validateDomainConfiguration(domain: string): Promise<boolean> {
    try {
      // Check CNAME records
      const records = await this.resolveCNAME(domain);
      const platformDomain = this.configService.get('PLATFORM_DOMAIN');
      
      return records.includes(platformDomain);
    } catch (error) {
      return false;
    }
  }
  
  async waitForDNSPropagation(domain: string, maxWaitMinutes: number = 30): Promise<boolean> {
    const maxAttempts = maxWaitMinutes * 2; // Check every 30 seconds
    
    for (let attempt = 0; attempt < maxAttempts; attempt++) {
      if (await this.validateDomainConfiguration(domain)) {
        return true;
      }
      
      await this.sleep(30000); // Wait 30 seconds
    }
    
    return false;
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## Performance Monitoring

### Application Metrics

#### Custom Metrics Collection
```typescript
@Injectable()
export class MetricsService {
  private readonly metrics = {
    requestDuration: new prometheus.Histogram({
      name: 'http_request_duration_seconds',
      help: 'Duration of HTTP requests in seconds',
      labelNames: ['method', 'route', 'status_code', 'tenant_id']
    }),
    
    videoProcessingDuration: new prometheus.Histogram({
      name: 'video_processing_duration_seconds',
      help: 'Duration of video processing in seconds',
      labelNames: ['quality', 'file_size_mb']
    }),
    
    cacheHitRate: new prometheus.Counter({
      name: 'cache_hits_total',
      help: 'Total cache hits',
      labelNames: ['cache_type', 'tenant_id']
    })
  };
  
  recordRequestDuration(
    method: string,
    route: string,
    statusCode: number,
    tenantId: string,
    duration: number
  ) {
    this.metrics.requestDuration
      .labels(method, route, statusCode.toString(), tenantId)
      .observe(duration);
  }
  
  recordVideoProcessing(quality: string, fileSizeMB: number, duration: number) {
    this.metrics.videoProcessingDuration
      .labels(quality, fileSizeMB.toString())
      .observe(duration);
  }
  
  recordCacheHit(cacheType: string, tenantId: string) {
    this.metrics.cacheHitRate
      .labels(cacheType, tenantId)
      .inc();
  }
}
```

#### Health Check Implementation
```typescript
@Controller('health')
export class HealthController {
  constructor(
    private databaseService: DatabaseService,
    private redisService: RedisService,
    private videoQueueService: VideoQueueService
  ) {}
  
  @Get()
  async checkHealth(): Promise<HealthStatus> {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkVideoQueue(),
      this.checkR2Storage()
    ]);
    
    const status: HealthStatus = {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      services: {}
    };
    
    checks.forEach((check, index) => {
      const serviceName = ['database', 'redis', 'video_queue', 'r2_storage'][index];
      
      if (check.status === 'fulfilled') {
        status.services[serviceName] = { status: 'healthy' };
      } else {
        status.status = 'unhealthy';
        status.services[serviceName] = {
          status: 'unhealthy',
          error: check.reason.message
        };
      }
    });
    
    return status;
  }
  
  private async checkDatabase(): Promise<void> {
    await this.databaseService.query('SELECT 1');
  }
  
  private async checkRedis(): Promise<void> {
    await this.redisService.ping();
  }
  
  private async checkVideoQueue(): Promise<void> {
    await this.videoQueueService.getQueueStats();
  }
  
  private async checkR2Storage(): Promise<void> {
    // Perform a simple HEAD request to R2
    await this.r2Client.headBucket({ Bucket: 'video-storage' });
  }
}
```

## Error Handling & Logging

### Centralized Error Handling
```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);
  
  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    
    let status = 500;
    let message = 'Internal server error';
    let code = 'INTERNAL_ERROR';
    
    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();
      message = typeof exceptionResponse === 'string' 
        ? exceptionResponse 
        : (exceptionResponse as any).message;
    }
    
    // Log error with context
    this.logger.error(
      `${request.method} ${request.url}`,
      {
        tenantId: request['tenant']?.id,
        userId: request['user']?.id,
        error: exception instanceof Error ? exception.stack : exception,
        requestId: request.headers['x-request-id']
      }
    );
    
    // Send error response
    response.status(status).json({
      error: {
        code,
        message,
        timestamp: new Date().toISOString(),
        path: request.url,
        requestId: request.headers['x-request-id']
      }
    });
  }
}
```

### Structured Logging
```typescript
@Injectable()
export class LoggerService {
  private readonly logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
      winston.format.timestamp(),
      winston.format.errors({ stack: true }),
      winston.format.json()
    ),
    transports: [
      new winston.transports.Console(),
      new winston.transports.CloudWatch({
        logGroupName: 'course-platform',
        logStreamName: 'application',
        region: 'us-east-1'
      })
    ]
  });
  
  logTenantAction(action: string, tenantId: string, metadata?: any) {
    this.logger.info('Tenant action', {
      action,
      tenantId,
      ...metadata
    });
  }
  
  logVideoProcessing(lectureId: string, status: string, metadata?: any) {
    this.logger.info('Video processing', {
      lectureId,
      status,
      ...metadata
    });
  }
  
  logSecurityEvent(event: string, userId: string, tenantId: string, metadata?: any) {
    this.logger.warn('Security event', {
      event,
      userId,
      tenantId,
      ...metadata
    });
  }
}
```

This comprehensive technical specification provides the detailed implementation guidance needed to build the multi-tenant online course platform with all the features and optimizations outlined in the system design document.