[Back to Index](./index.md)

# API Integration

## Complete API Service Implementation

The API service provides a type-safe wrapper around Firebase Cloud Functions with automatic retry logic, offline queue support, and error handling.

### API Service

```typescript
// libs/mobile/data-access/src/lib/services/api.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpHeaders, HttpErrorResponse } from '@angular/common/http';
import { Functions, httpsCallable, httpsCallableData } from '@angular/fire/functions';
import { Auth } from '@angular/fire/auth';
import { Observable, from, throwError, of, timer } from 'rxjs';
import { catchError, retry, timeout, switchMap, map } from 'rxjs/operators';
import { NetworkService } from './network.service';
import { SyncQueueService } from './sync-queue.service';

// Type-safe function names
export type CloudFunctionName =
  | 'allocateSequence'
  | 'generatePDF'
  | 'processVoiceCommand'
  | 'exportReport';

// Request/Response interfaces
export interface AllocateSequenceRequest {
  tenantId: string;
  sequenceType: 'jobNumber' | 'vehicleNumber' | 'machineNumber' | 'teamMemberNumber' | 'ordinalNumber';
  jobId?: string; // Required for ordinalNumber
}

export interface AllocateSequenceResponse {
  number: number;
  sequenceType: string;
  allocatedAt: string;
}

export interface GeneratePDFRequest {
  jobId: string;
  options?: {
    includePhotos?: boolean;
    includeSignatures?: boolean;
    templateId?: string;
  };
}

export interface ProcessVoiceCommandRequest {
  transcript: string;
  context: {
    activeJobId?: string;
    currentScreen?: string;
    previousCommands?: string[];
    tenantId: string;
    userId: string;
  };
}

export interface ProcessVoiceCommandResponse {
  intent: string;
  action: {
    type: string;
    payload: any;
  };
  response: string;
  confidence: number;
}

export interface ApiResponse<T = any> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  metadata?: {
    timestamp: number;
    requestId: string;
  };
}

@Injectable({ providedIn: 'root' })
export class ApiService {
  private readonly http = inject(HttpClient);
  private readonly functions = inject(Functions);
  private readonly auth = inject(Auth);
  private readonly network = inject(NetworkService);
  private readonly syncQueue = inject(SyncQueueService);

  // Cloud Functions base configuration
  private readonly functionsConfig = {
    region: 'us-central1',
    timeout: 30000, // 30 seconds
    retryConfig: {
      count: 3,
      delay: 1000,
      maxDelay: 5000
    }
  };

  /**
   * Call a Cloud Function with automatic retry and offline queue
   */
  callFunction<TRequest, TResponse>(
    functionName: CloudFunctionName,
    data: TRequest,
    options?: {
      requiresAuth?: boolean;
      queueIfOffline?: boolean;
      timeout?: number;
    }
  ): Observable<TResponse> {
    const opts = {
      requiresAuth: true,
      queueIfOffline: false,
      timeout: this.functionsConfig.timeout,
      ...options
    };

    // Check if offline and should queue
    if (!this.network.isOnline() && opts.queueIfOffline) {
      return this.queueFunctionCall(functionName, data);
    }

    // Get the callable function
    const callable = httpsCallable<TRequest, TResponse>(
      this.functions,
      functionName,
      { timeout: opts.timeout }
    );

    return from(callable(data)).pipe(
      retry({
        count: this.functionsConfig.retryConfig.count,
        delay: this.exponentialBackoff
      }),
      catchError(error => this.handleFunctionError(error, functionName))
    );
  }

  /**
   * Queue function call for later execution when online
   */
  private queueFunctionCall<T>(functionName: string, data: any): Observable<T> {
    const operation = {
      type: 'CLOUD_FUNCTION',
      payload: {
        functionName,
        data,
        timestamp: Date.now()
      },
      timestamp: Date.now()
    };

    this.syncQueue.addOperation(operation);

    // Return optimistic response or pending indicator
    return of({
      success: true,
      data: null,
      metadata: {
        queued: true,
        queuedAt: Date.now()
      }
    } as any);
  }

  /**
   * Exponential backoff for retries
   */
  private exponentialBackoff(error: any, retryCount: number): Observable<number> {
    const delay = Math.min(
      this.functionsConfig.retryConfig.delay * Math.pow(2, retryCount - 1),
      this.functionsConfig.retryConfig.maxDelay
    );
    console.warn(`Retry attempt ${retryCount} after ${delay}ms for error:`, error);
    return timer(delay);
  }

  /**
   * Handle Cloud Function errors
   */
  private handleFunctionError(error: any, functionName: string): Observable<never> {
    let errorMessage = 'An unexpected error occurred';
    let errorCode = 'UNKNOWN_ERROR';

    if (error.code) {
      // Firebase Function error codes
      switch (error.code) {
        case 'functions/cancelled':
          errorMessage = 'Request was cancelled';
          errorCode = 'CANCELLED';
          break;
        case 'functions/unknown':
          errorMessage = error.message || 'Unknown error occurred';
          errorCode = 'UNKNOWN';
          break;
        case 'functions/deadline-exceeded':
          errorMessage = 'Request timeout';
          errorCode = 'TIMEOUT';
          break;
        case 'functions/permission-denied':
          errorMessage = 'Permission denied';
          errorCode = 'PERMISSION_DENIED';
          break;
        case 'functions/unauthenticated':
          errorMessage = 'Authentication required';
          errorCode = 'UNAUTHENTICATED';
          break;
        default:
          errorMessage = error.message || errorMessage;
          errorCode = error.code;
      }
    }

    console.error(`Cloud Function ${functionName} error:`, {
      code: errorCode,
      message: errorMessage,
      details: error
    });

    return throwError(() => ({
      code: errorCode,
      message: errorMessage,
      functionName,
      timestamp: Date.now()
    }));
  }

  // ============================================
  // Specific Cloud Function Implementations
  // ============================================

  /**
   * Allocate next sequence number for tenant
   */
  allocateSequence(request: AllocateSequenceRequest): Observable<AllocateSequenceResponse> {
    return this.callFunction<AllocateSequenceRequest, AllocateSequenceResponse>(
      'allocateSequence',
      request,
      { queueIfOffline: true } // Queue this if offline
    );
  }

  /**
   * Generate PDF report (Phase 2)
   */
  generatePDF(jobId: string, options?: {
    includePhotos?: boolean;
    includeSignatures?: boolean;
  }): Observable<{ url: string; expiresAt: string }> {
    return this.callFunction<GeneratePDFRequest, { url: string; expiresAt: string }>(
      'generatePDF',
      { jobId, options },
      {
        timeout: 60000, // 60 seconds for PDF generation
        queueIfOffline: false // Don't queue PDF generation
      }
    );
  }

  /**
   * Process voice command through LLM
   */
  processVoiceCommand(command: {
    transcript: string;
    context: {
      activeJobId?: string;
      currentScreen?: string;
      previousCommands?: string[];
    };
  }): Observable<ProcessVoiceCommandResponse> {
    const tenantId = localStorage.getItem('tenantId') || '';
    const userId = this.auth.currentUser?.uid || '';

    return this.callFunction<ProcessVoiceCommandRequest, ProcessVoiceCommandResponse>(
      'processVoiceCommand',
      {
        ...command,
        context: {
          ...command.context,
          tenantId,
          userId
        }
      },
      {
        timeout: 15000, // 15 seconds for voice processing
        queueIfOffline: false // Real-time only
      }
    );
  }

  /**
   * Export report
   */
  exportReport(params: {
    jobId?: string;
    dateRange?: { start: Date; end: Date };
    format: 'csv' | 'xlsx' | 'pdf';
  }): Observable<{ url: string }> {
    return this.callFunction(
      'exportReport',
      params,
      {
        timeout: 45000,
        queueIfOffline: false
      }
    );
  }
}
```

## API Client Configuration (HTTP Interceptors)

### Auth Interceptor

```typescript
// libs/mobile/data-access/src/lib/interceptors/auth.interceptor.ts
import { Injectable, inject } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError, from } from 'rxjs';
import { catchError, switchMap } from 'rxjs/operators';
import { Auth } from '@angular/fire/auth';
import { Router } from '@angular/router';
import { Capacitor } from '@capacitor/core';
import { environment } from '@environments/environment';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  private readonly auth = inject(Auth);
  private readonly router = inject(Router);

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Skip auth for public endpoints
    if (this.isPublicEndpoint(req.url)) {
      return next.handle(req);
    }

    // Add auth token to requests
    return from(this.auth.currentUser?.getIdToken() ?? Promise.resolve(null)).pipe(
      switchMap(token => {
        if (!token) {
          // No token available, redirect to login
          this.router.navigate(['/auth/login']);
          return throwError(() => new Error('No authentication token'));
        }

        // Clone request with auth header
        const authReq = req.clone({
          setHeaders: {
            Authorization: `Bearer ${token}`,
            'X-Tenant-Id': this.getTenantId(),
            'X-Client-Version': this.getAppVersion(),
            'X-Platform': this.getPlatform()
          }
        });

        return next.handle(authReq);
      }),
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          // Token expired or invalid
          this.handleAuthError();
        }
        return throwError(() => error);
      })
    );
  }

  private isPublicEndpoint(url: string): boolean {
    const publicEndpoints = ['/api/public', '/api/health'];
    return publicEndpoints.some(endpoint => url.includes(endpoint));
  }

  private handleAuthError(): void {
    // Sign out and redirect to login
    this.auth.signOut();
    this.router.navigate(['/auth/login']);
  }

  private getTenantId(): string {
    // Get from user claims or local storage
    return localStorage.getItem('tenantId') || '';
  }

  private getAppVersion(): string {
    return environment.appVersion;
  }

  private getPlatform(): string {
    return Capacitor.getPlatform(); // 'ios', 'android', or 'web'
  }
}
```

### Offline Interceptor

```typescript
// libs/mobile/data-access/src/lib/interceptors/offline.interceptor.ts
import { Injectable, inject } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent, HttpResponse } from '@angular/common/http';
import { Observable, throwError, of } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { NetworkService } from '../services/network.service';
import { SyncQueueService } from '../services/sync-queue.service';

@Injectable()
export class OfflineInterceptor implements HttpInterceptor {
  private readonly network = inject(NetworkService);
  private readonly syncQueue = inject(SyncQueueService);

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Check if we're offline before making request
    if (!this.network.isOnline()) {
      // Check if this request can be queued
      if (this.canQueueRequest(req)) {
        return this.queueRequest(req);
      }

      // Return offline error
      return throwError(() => ({
        status: 0,
        statusText: 'Offline',
        error: {
          message: 'No network connection available',
          code: 'OFFLINE'
        }
      }));
    }

    return next.handle(req).pipe(
      catchError(error => {
        // Network error occurred during request
        if (error.status === 0 && this.canQueueRequest(req)) {
          return this.queueRequest(req);
        }
        return throwError(() => error);
      })
    );
  }

  private canQueueRequest(req: HttpRequest<any>): boolean {
    // Only queue POST/PUT/PATCH requests
    const queuableMethods = ['POST', 'PUT', 'PATCH'];
    return queuableMethods.includes(req.method) && !req.url.includes('auth');
  }

  private queueRequest(req: HttpRequest<any>): Observable<HttpEvent<any>> {
    // Add to sync queue
    this.syncQueue.addOperation({
      type: 'HTTP_REQUEST',
      payload: {
        url: req.url,
        method: req.method,
        body: req.body,
        headers: req.headers.keys().reduce((acc, key) => ({
          ...acc,
          [key]: req.headers.get(key)
        }), {})
      },
      timestamp: Date.now()
    });

    // Return success response for optimistic update
    return of(new HttpResponse({
      status: 202,
      statusText: 'Accepted',
      body: {
        queued: true,
        message: 'Request queued for sync when online'
      }
    }));
  }
}
```

### Interceptor Configuration

```typescript
// app.config.ts - Configure interceptors
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptorsFromDi } from '@angular/common/http';
import { HTTP_INTERCEPTORS } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptorsFromDi()),
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: OfflineInterceptor,
      multi: true
    },
    // ... other providers
  ]
};
```

## Usage Examples

### Calling Cloud Functions

```typescript
// In a component or service
import { inject } from '@angular/core';
import { ApiService } from '@libs/mobile/data-access';

export class JobsService {
  private readonly api = inject(ApiService);

  allocateJobNumber(tenantId: string): Observable<number> {
    return this.api.allocateSequence({
      tenantId,
      sequenceType: 'jobNumber'
    }).pipe(
      map(response => response.number)
    );
  }

  processVoice(transcript: string, activeJobId?: string): Observable<any> {
    return this.api.processVoiceCommand({
      transcript,
      context: {
        activeJobId,
        currentScreen: 'jobs-list'
      }
    });
  }
}
```

## Error Handling Patterns

```typescript
// Handling API errors in components
export class JobDetailComponent {
  private readonly api = inject(ApiService);
  private readonly toastController = inject(ToastController);

  protected saveJob(job: Job): void {
    this.api.callFunction('updateJob', job).pipe(
      catchError(async (error) => {
        await this.showError(error.message);
        return of(null);
      })
    ).subscribe(result => {
      if (result) {
        this.showSuccess('Job saved successfully');
      }
    });
  }

  private async showError(message: string): Promise<void> {
    const toast = await this.toastController.create({
      message,
      duration: 3000,
      color: 'danger'
    });
    await toast.present();
  }
}
```

---

[← Previous: State Management](./state-management.md) | [Back to Index](./index.md) | [Next: Routing →](./routing.md)
