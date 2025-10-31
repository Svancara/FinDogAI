[Back to Index](./index.md)

# Performance Monitoring Architecture

## Overview

Performance monitoring for FinDogAI presents unique challenges due to its mobile-first, offline-first architecture. Traditional real-time monitoring assumptions break down when users operate in intermittent connectivity environments. This architecture addresses these constraints with a **deferred, sampled, and context-aware** monitoring approach.

## Core Principles

### 1. Offline-First Reality

**Challenge**: Performance metrics collected offline may not reach the monitoring system for hours or days, if ever.

**Response**:
- Accept that monitoring will be **incomplete and delayed**
- Focus on **trends over absolute coverage**
- Design for **eventual consistency** in metrics collection
- Prioritize **critical metrics** that inform actionable decisions

### 2. Sparse & Selective Collection

**Challenge**: Excessive telemetry drains battery and consumes mobile data budgets.

**Response**:
- **Sample aggressively** (1-10% of operations)
- **Prioritize critical paths** (voice commands, sync operations)
- **Budget-aware transmission** (WiFi-only by default)
- **Local aggregation** before transmission

### 3. Context-Aware Metrics

**Challenge**: Raw metrics without context (network state, device capabilities) are misleading.

**Response**:
- Tag all metrics with **connectivity state** (online/offline/slow-2g/3g/4g/wifi)
- Include **device context** (platform, memory, battery level)
- Track **sync queue depth** as performance indicator
- Correlate performance with **user workflows**

## Monitoring Architecture

### Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│                    Client (Mobile App)                   │
├─────────────────────────────────────────────────────────┤
│  1. Metrics Collection Service (in-memory aggregation)   │
│  2. Local Storage Buffer (IndexedDB, 7-day retention)    │
│  3. Transmission Service (WiFi-only, batched uploads)    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼ (WiFi + sampling)
┌─────────────────────────────────────────────────────────┐
│              Cloud Infrastructure (Firebase)             │
├─────────────────────────────────────────────────────────┤
│  4. Ingestion Function (HTTPS callable, throttled)       │
│  5. Firestore Metrics Collection (tenant-isolated)       │
│  6. Analytics Pipeline (BigQuery export, scheduled)      │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Analysis & Alerting (External)              │
├─────────────────────────────────────────────────────────┤
│  7. BigQuery Analysis (weekly reports)                   │
│  8. Dashboards (Google Data Studio / Grafana)            │
│  9. Alerts (regression detection, error spikes)          │
└─────────────────────────────────────────────────────────┘
```

## Metrics Collection Strategy

### Critical Metrics (10% Sampling)

These metrics directly impact user experience and are worth the telemetry cost:

| Metric | What It Measures | Why It Matters | Collection Trigger |
|--------|------------------|----------------|-------------------|
| **Voice Command Latency** | Time from wake-word to TTS response | Core UX, users notice >3s | Every voice command |
| **Sync Queue Processing Time** | Duration to clear pending operations | Offline→online transition smoothness | When queue processes |
| **App Cold Start Time** | Time to interactive after launch | First impression, abandonment risk | First launch per session |
| **Firestore Read/Write Latency** | Roundtrip time for DB operations | Online performance baseline | 10% sample of operations |
| **Crash Rate** | App crashes per session | Stability indicator | Every crash (100%) |

### Secondary Metrics (1% Sampling)

Useful for diagnosis but not critical for day-to-day monitoring:

| Metric | What It Measures | Sampling Rate |
|--------|------------------|---------------|
| Component Render Time | Angular change detection cycles | 1% |
| Bundle Load Time | Lazy-loaded module timing | 1% |
| Memory Usage | Heap size over time | 1% samples/hour |
| Network Request Size | API payload sizes | 1% |
| IndexedDB Query Time | Local data access speed | 1% |

### Contextual Tags (All Metrics)

Every metric includes these tags for segmentation:

```typescript
interface MetricContext {
  // Connectivity context
  networkState: 'online' | 'offline' | 'slow-2g' | '2g' | '3g' | '4g' | 'wifi';
  syncQueueDepth: number;  // Pending operations count

  // Device context
  platform: 'ios' | 'android' | 'web';
  deviceMemory?: number;    // GB, if available
  batteryLevel?: number;    // 0-100, if available

  // User context
  tenantId: string;         // For tenant-level analysis
  userId: string;           // Hashed for privacy
  sessionId: string;        // Unique per app session

  // App context
  appVersion: string;
  buildNumber: string;

  // Timestamp (client-side)
  timestamp: number;        // Unix epoch ms
}
```

## Implementation

### 1. Metrics Collection Service

```typescript
// ABOUTME: Core performance monitoring service with aggressive sampling and offline buffering
// libs/mobile/data-access/src/lib/services/performance-monitoring.service.ts

import { Injectable, inject } from '@angular/core';
import { NetworkService } from './network.service';
import { SyncQueueService } from './sync-queue.service';
import { Platform } from '@ionic/angular';

export interface PerformanceMetric {
  name: string;
  value: number;
  unit: 'ms' | 'bytes' | 'count';
  context: MetricContext;
  tags?: Record<string, string>;
}

interface MetricContext {
  networkState: string;
  syncQueueDepth: number;
  platform: string;
  deviceMemory?: number;
  batteryLevel?: number;
  tenantId: string;
  userId: string;
  sessionId: string;
  appVersion: string;
  buildNumber: string;
  timestamp: number;
}

@Injectable({ providedIn: 'root' })
export class PerformanceMonitoringService {
  private readonly network = inject(NetworkService);
  private readonly syncQueue = inject(SyncQueueService);
  private readonly platform = inject(Platform);

  private readonly sessionId = crypto.randomUUID();
  private metricsBuffer: PerformanceMetric[] = [];
  private readonly MAX_BUFFER_SIZE = 500;
  private readonly FLUSH_INTERVAL_MS = 60000; // 1 minute

  // Sampling rates
  private readonly CRITICAL_SAMPLE_RATE = 0.1;  // 10%
  private readonly SECONDARY_SAMPLE_RATE = 0.01; // 1%

  constructor() {
    // Periodic flush to IndexedDB
    setInterval(() => this.flushToLocalStorage(), this.FLUSH_INTERVAL_MS);

    // Flush and upload when online + WiFi
    this.network.online$.pipe(
      filter(online => online && this.isWiFiConnection())
    ).subscribe(() => this.uploadMetrics());
  }

  /**
   * Record a critical metric (10% sampling)
   */
  recordCritical(name: string, value: number, unit: 'ms' | 'bytes' | 'count', tags?: Record<string, string>): void {
    if (Math.random() > this.CRITICAL_SAMPLE_RATE) return;
    this.recordMetric(name, value, unit, tags);
  }

  /**
   * Record a secondary metric (1% sampling)
   */
  recordSecondary(name: string, value: number, unit: 'ms' | 'bytes' | 'count', tags?: Record<string, string>): void {
    if (Math.random() > this.SECONDARY_SAMPLE_RATE) return;
    this.recordMetric(name, value, unit, tags);
  }

  /**
   * Record a metric without sampling (use sparingly!)
   */
  recordAlways(name: string, value: number, unit: 'ms' | 'bytes' | 'count', tags?: Record<string, string>): void {
    this.recordMetric(name, value, unit, tags);
  }

  private recordMetric(name: string, value: number, unit: 'ms' | 'bytes' | 'count', tags?: Record<string, string>): void {
    const metric: PerformanceMetric = {
      name,
      value,
      unit,
      context: this.buildContext(),
      tags
    };

    this.metricsBuffer.push(metric);

    // Flush if buffer is full
    if (this.metricsBuffer.length >= this.MAX_BUFFER_SIZE) {
      this.flushToLocalStorage();
    }
  }

  private buildContext(): MetricContext {
    const context: MetricContext = {
      networkState: this.network.getConnectionType(),
      syncQueueDepth: this.getSyncQueueDepth(),
      platform: this.platform.is('ios') ? 'ios' : this.platform.is('android') ? 'android' : 'web',
      tenantId: this.getTenantId(),
      userId: this.getUserIdHash(),
      sessionId: this.sessionId,
      appVersion: this.getAppVersion(),
      buildNumber: this.getBuildNumber(),
      timestamp: Date.now()
    };

    // Add device memory if available
    if ('deviceMemory' in navigator) {
      context.deviceMemory = (navigator as any).deviceMemory;
    }

    // Add battery level if available (async, best effort)
    this.getBatteryLevel().then(level => {
      if (level !== null) {
        context.batteryLevel = level;
      }
    });

    return context;
  }

  private async getBatteryLevel(): Promise<number | null> {
    if (!('getBattery' in navigator)) return null;

    try {
      const battery = await (navigator as any).getBattery();
      return Math.round(battery.level * 100);
    } catch {
      return null;
    }
  }

  private isWiFiConnection(): boolean {
    const connection = (navigator as any).connection;
    return connection?.type === 'wifi' || connection?.effectiveType === '4g';
  }

  private getSyncQueueDepth(): number {
    // Implement based on your SyncQueueService
    let depth = 0;
    this.syncQueue.pendingCount$.pipe(take(1)).subscribe(count => depth = count);
    return depth;
  }

  private getTenantId(): string {
    // Retrieve from auth state or store
    return 'tenant-id'; // Placeholder
  }

  private getUserIdHash(): string {
    // Hash user ID for privacy
    return 'user-id-hash'; // Placeholder
  }

  private getAppVersion(): string {
    return '1.0.0'; // From environment or package.json
  }

  private getBuildNumber(): string {
    return '100'; // From build config
  }

  private async flushToLocalStorage(): Promise<void> {
    if (this.metricsBuffer.length === 0) return;

    const metrics = [...this.metricsBuffer];
    this.metricsBuffer = [];

    try {
      const db = await this.openMetricsDB();
      const tx = db.transaction('metrics', 'readwrite');
      const store = tx.objectStore('metrics');

      for (const metric of metrics) {
        await store.add(metric);
      }

      await tx.complete;

      // Clean up old metrics (>7 days)
      await this.cleanupOldMetrics(db);
    } catch (error) {
      console.error('Failed to flush metrics to IndexedDB:', error);
    }
  }

  private async uploadMetrics(): Promise<void> {
    // Only upload on WiFi
    if (!this.isWiFiConnection()) return;

    try {
      const db = await this.openMetricsDB();
      const metrics = await db.getAll('metrics');

      if (metrics.length === 0) return;

      // Upload in batches of 100
      const batches = this.chunkArray(metrics, 100);

      for (const batch of batches) {
        await this.sendBatch(batch);

        // Remove uploaded metrics from IndexedDB
        const tx = db.transaction('metrics', 'readwrite');
        for (const metric of batch) {
          await tx.objectStore('metrics').delete(metric.timestamp);
        }
        await tx.complete;
      }
    } catch (error) {
      console.error('Failed to upload metrics:', error);
      // Metrics remain in IndexedDB for retry
    }
  }

  private async sendBatch(metrics: PerformanceMetric[]): Promise<void> {
    // Call Cloud Function to ingest metrics
    const functions = getFunctions();
    const ingestMetrics = httpsCallable(functions, 'ingestMetrics');

    await ingestMetrics({ metrics });
  }

  private chunkArray<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }

  private async openMetricsDB(): Promise<IDBDatabase> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open('FinDogMetrics', 1);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => resolve(request.result);

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;
        if (!db.objectStoreNames.contains('metrics')) {
          db.createObjectStore('metrics', { keyPath: 'timestamp' });
        }
      };
    });
  }

  private async cleanupOldMetrics(db: IDBDatabase): Promise<void> {
    const sevenDaysAgo = Date.now() - (7 * 24 * 60 * 60 * 1000);

    const tx = db.transaction('metrics', 'readwrite');
    const store = tx.objectStore('metrics');
    const allMetrics = await store.getAll();

    for (const metric of allMetrics) {
      if (metric.timestamp < sevenDaysAgo) {
        await store.delete(metric.timestamp);
      }
    }

    await tx.complete;
  }
}
```

### 2. Usage Examples

#### Voice Command Latency

```typescript
// features/voice/services/voice-command.service.ts
export class VoiceCommandService {
  private readonly perfMon = inject(PerformanceMonitoringService);

  async executeCommand(audioBlob: Blob): Promise<VoiceCommandResult> {
    const startTime = performance.now();

    try {
      // STT
      const sttStart = performance.now();
      const transcript = await this.sttService.transcribe(audioBlob);
      this.perfMon.recordCritical('voice.stt.latency', performance.now() - sttStart, 'ms');

      // NLU
      const nluStart = performance.now();
      const intent = await this.nluService.parse(transcript);
      this.perfMon.recordCritical('voice.nlu.latency', performance.now() - nluStart, 'ms');

      // TTS
      const ttsStart = performance.now();
      await this.ttsService.speak(intent.confirmation);
      this.perfMon.recordCritical('voice.tts.latency', performance.now() - ttsStart, 'ms');

      // End-to-end
      this.perfMon.recordCritical('voice.e2e.latency', performance.now() - startTime, 'ms', {
        command: intent.type
      });

      return { success: true, intent };
    } catch (error) {
      this.perfMon.recordAlways('voice.error', 1, 'count', {
        error: error.message,
        stage: this.getFailureStage(error)
      });
      throw error;
    }
  }
}
```

#### Sync Queue Processing

```typescript
// services/sync-queue.service.ts
export class SyncQueueService {
  private readonly perfMon = inject(PerformanceMonitoringService);

  async processPendingOperations(): Promise<void> {
    const startTime = performance.now();
    const queueDepth = this.queue.value.length;

    if (queueDepth === 0) return;

    let successCount = 0;
    let failureCount = 0;

    for (const op of this.queue.value) {
      try {
        await this.processOperation(op);
        this.removeOperation(op.id!);
        successCount++;
      } catch (error) {
        failureCount++;
        this.handleOperationError(op);
      }
    }

    // Record sync performance
    this.perfMon.recordCritical('sync.processing.time', performance.now() - startTime, 'ms', {
      queueDepth: queueDepth.toString(),
      successCount: successCount.toString(),
      failureCount: failureCount.toString()
    });
  }
}
```

#### App Cold Start

```typescript
// app.component.ts
export class AppComponent implements OnInit {
  private readonly perfMon = inject(PerformanceMonitoringService);

  ngOnInit() {
    // Mark app ready
    const navStart = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming;
    const timeToInteractive = performance.now();

    this.perfMon.recordCritical('app.cold_start', timeToInteractive, 'ms', {
      domContentLoaded: navStart.domContentLoadedEventEnd.toString(),
      domComplete: navStart.domComplete.toString()
    });
  }
}
```

### 3. Cloud Ingestion Function

```typescript
// ABOUTME: Cloud Function for ingesting performance metrics from mobile clients
// functions/src/ingest-metrics.ts

import * as functions from 'firebase-functions';
import { getFirestore, FieldValue } from 'firebase-admin/firestore';

interface MetricPayload {
  metrics: PerformanceMetric[];
}

export const ingestMetrics = functions
  .region('europe-west1')
  .runWith({
    timeoutSeconds: 60,
    memory: '256MB'
  })
  .https.onCall(async (data: MetricPayload, context) => {
    // Require authentication
    if (!context.auth) {
      throw new functions.https.HttpsError('unauthenticated', 'Must be authenticated');
    }

    const { metrics } = data;

    if (!metrics || !Array.isArray(metrics) || metrics.length === 0) {
      throw new functions.https.HttpsError('invalid-argument', 'Invalid metrics payload');
    }

    // Throttle: Max 1000 metrics per call
    if (metrics.length > 1000) {
      throw new functions.https.HttpsError('invalid-argument', 'Too many metrics');
    }

    const db = getFirestore();
    const batch = db.batch();

    // Group metrics by tenant for isolation
    const metricsByTenant = groupBy(metrics, m => m.context.tenantId);

    for (const [tenantId, tenantMetrics] of Object.entries(metricsByTenant)) {
      // Store aggregated metrics per tenant
      const metricsRef = db
        .collection('tenants')
        .doc(tenantId)
        .collection('performance_metrics')
        .doc();

      batch.set(metricsRef, {
        metrics: tenantMetrics,
        receivedAt: FieldValue.serverTimestamp(),
        count: tenantMetrics.length,
        userId: context.auth.uid
      });
    }

    await batch.commit();

    return { success: true, processed: metrics.length };
  });

function groupBy<T>(array: T[], keyGetter: (item: T) => string): Record<string, T[]> {
  return array.reduce((result, item) => {
    const key = keyGetter(item);
    if (!result[key]) {
      result[key] = [];
    }
    result[key].push(item);
    return result;
  }, {} as Record<string, T[]>);
}
```

### 4. BigQuery Export (Scheduled)

```typescript
// ABOUTME: Scheduled function to export metrics to BigQuery for analysis
// functions/src/export-metrics.ts

import * as functions from 'firebase-functions';
import { getFirestore } from 'firebase-admin/firestore';
import { BigQuery } from '@google-cloud/bigquery';

export const exportMetricsToBigQuery = functions
  .region('europe-west1')
  .pubsub
  .schedule('0 2 * * *') // Daily at 2 AM UTC
  .onRun(async () => {
    const db = getFirestore();
    const bigquery = new BigQuery();
    const dataset = bigquery.dataset('findogai_metrics');
    const table = dataset.table('performance_metrics');

    // Get all tenants
    const tenantsSnapshot = await db.collection('tenants').get();

    for (const tenantDoc of tenantsSnapshot.docs) {
      const tenantId = tenantDoc.id;

      // Get metrics from last 24 hours
      const yesterday = new Date();
      yesterday.setDate(yesterday.getDate() - 1);

      const metricsSnapshot = await db
        .collection('tenants')
        .doc(tenantId)
        .collection('performance_metrics')
        .where('receivedAt', '>=', yesterday)
        .get();

      if (metricsSnapshot.empty) continue;

      // Flatten metrics for BigQuery
      const rows = metricsSnapshot.docs.flatMap(doc => {
        const data = doc.data();
        return data.metrics.map((metric: any) => ({
          tenant_id: tenantId,
          metric_name: metric.name,
          metric_value: metric.value,
          metric_unit: metric.unit,
          network_state: metric.context.networkState,
          sync_queue_depth: metric.context.syncQueueDepth,
          platform: metric.context.platform,
          app_version: metric.context.appVersion,
          timestamp: new Date(metric.context.timestamp),
          received_at: data.receivedAt.toDate(),
          tags: JSON.stringify(metric.tags || {})
        }));
      });

      // Insert into BigQuery
      await table.insert(rows);

      // Clean up Firestore after export
      const batch = db.batch();
      metricsSnapshot.docs.forEach(doc => batch.delete(doc.ref));
      await batch.commit();
    }

    console.log('Metrics exported to BigQuery successfully');
  });
```

## Analysis & Dashboards

### Key Questions to Answer

1. **Is performance acceptable?**
   - P50, P95, P99 latencies for critical operations
   - Segmented by network state (WiFi vs cellular vs offline)

2. **Are users experiencing degradation?**
   - Week-over-week trends
   - Correlation with app version releases

3. **Where should we optimize?**
   - Slowest operations by percentile
   - Operations with highest variance

4. **How does offline-first work in practice?**
   - Sync queue depth distribution
   - Time to clear queue after reconnection
   - Conflict resolution frequency

### Sample BigQuery Queries

#### Voice Command Latency by Network State

```sql
SELECT
  network_state,
  APPROX_QUANTILES(metric_value, 100)[OFFSET(50)] AS p50_ms,
  APPROX_QUANTILES(metric_value, 100)[OFFSET(95)] AS p95_ms,
  APPROX_QUANTILES(metric_value, 100)[OFFSET(99)] AS p99_ms,
  COUNT(*) AS sample_count
FROM `findogai_metrics.performance_metrics`
WHERE
  metric_name = 'voice.e2e.latency'
  AND timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY network_state
ORDER BY p95_ms DESC;
```

#### Sync Queue Processing Time Trends

```sql
SELECT
  DATE(timestamp) AS date,
  AVG(metric_value) AS avg_processing_time_ms,
  APPROX_QUANTILES(metric_value, 100)[OFFSET(95)] AS p95_processing_time_ms,
  AVG(CAST(JSON_EXTRACT_SCALAR(tags, '$.queueDepth') AS INT64)) AS avg_queue_depth
FROM `findogai_metrics.performance_metrics`
WHERE
  metric_name = 'sync.processing.time'
  AND timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY date
ORDER BY date;
```

#### App Cold Start by Platform

```sql
SELECT
  platform,
  app_version,
  APPROX_QUANTILES(metric_value, 100)[OFFSET(50)] AS p50_ms,
  APPROX_QUANTILES(metric_value, 100)[OFFSET(90)] AS p90_ms,
  COUNT(*) AS sample_count
FROM `findogai_metrics.performance_metrics`
WHERE
  metric_name = 'app.cold_start'
  AND timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY platform, app_version
ORDER BY platform, app_version;
```

## Alerting Strategy

### Critical Alerts (Immediate Action)

| Alert | Condition | Threshold | Action |
|-------|-----------|-----------|--------|
| **Crash Rate Spike** | Crashes per session > threshold | >5% | Immediate investigation |
| **Voice Latency Degradation** | P95 voice.e2e.latency increases >50% | Week-over-week | Check API providers |
| **Sync Failures** | Sync error rate > threshold | >10% | Check backend health |

### Warning Alerts (Monitor)

| Alert | Condition | Threshold | Action |
|-------|-----------|-----------|--------|
| **Cold Start Degradation** | P90 cold start time increases | >20% WoW | Profile bundle size |
| **Memory Growth** | Memory usage trending up | Linear increase | Check for leaks |
| **Battery Drain** | Battery consumption rate high | Device-dependent | Profile background tasks |

## Privacy & Compliance

### Data Minimization

- **No PII in metrics**: User IDs are hashed (SHA-256)
- **No sensitive business data**: Financial amounts, job details excluded
- **Aggregated storage**: Raw metrics deleted after BigQuery export (7-day retention in Firestore)

### GDPR Compliance

- **Right to Erasure**: Delete tenant-specific metrics on request
- **Data Residency**: All data stored in `europe-west1` region
- **Transparency**: Document what metrics are collected in privacy policy

### User Control

```typescript
// Allow users to opt out of telemetry
export class SettingsComponent {
  telemetryEnabled = signal(true);

  toggleTelemetry(enabled: boolean): void {
    this.telemetryEnabled.set(enabled);
    localStorage.setItem('telemetry-enabled', enabled.toString());

    if (!enabled) {
      // Clear local metrics buffer
      this.perfMon.clearAllMetrics();
    }
  }
}
```

## Cost Estimation

### Telemetry Cost Breakdown

Assumptions:
- 100 active users
- 5 sessions per user per day
- 10 critical metrics per session (10% sampling)
- 50 secondary metrics per session (1% sampling)
- ~15 metrics collected per session

**Monthly Metrics Volume**: 100 users × 5 sessions/day × 15 metrics × 30 days = **225,000 metrics/month**

| Component | Volume | Cost |
|-----------|--------|------|
| **Cloud Function Invocations** | ~2,250 calls/month (batches of 100) | $0 (free tier: 2M) |
| **Firestore Writes** | 225,000 writes | $0.54 (7-day retention) |
| **BigQuery Ingestion** | ~50 MB/month | $0 (free tier: 10 GB) |
| **BigQuery Storage** | ~500 MB/year | $0.01/month |
| **BigQuery Queries** | ~1 TB/month processed | $5/month |
| **Total** | | **~$5.55/month** |

**Cost per user**: $0.055/month — negligible for MVP.

## Limitations & Caveats

### What This Architecture CANNOT Do

1. **Real-time alerting**: Metrics arrive hours/days late in offline scenarios
2. **Complete coverage**: Sampling means <10% of operations are measured
3. **User-level debugging**: Privacy-preserving hashing prevents user identification
4. **Session replay**: No video/interaction recording (too expensive for mobile)
5. **Network-level diagnostics**: Cannot measure ISP-level issues

### What This Architecture CAN Do

1. **Trend detection**: Week-over-week performance changes
2. **Regression identification**: App version comparisons
3. **Optimization prioritization**: Find slowest operations
4. **User experience validation**: Measure real-world latencies
5. **Capacity planning**: Understand sync queue patterns

## Monitoring Control & Kill Switches

Performance monitoring can be disabled at multiple levels to support development, testing, privacy compliance, and emergency scenarios.

### 1. User-Level Control (Privacy Preference)

Allow users to opt out of telemetry collection in app settings.

```typescript
// ABOUTME: User-facing telemetry preference management
// features/settings/services/telemetry-settings.service.ts

import { Injectable, inject, signal } from '@angular/core';
import { PerformanceMonitoringService } from '@core/services/performance-monitoring.service';

@Injectable({ providedIn: 'root' })
export class TelemetrySettingsService {
  private readonly perfMon = inject(PerformanceMonitoringService);

  // Persist preference in localStorage
  private readonly STORAGE_KEY = 'telemetry-enabled';

  readonly telemetryEnabled = signal<boolean>(this.loadPreference());

  constructor() {
    // Apply preference on startup
    this.applyPreference(this.telemetryEnabled());
  }

  toggleTelemetry(enabled: boolean): void {
    this.telemetryEnabled.set(enabled);
    localStorage.setItem(this.STORAGE_KEY, enabled.toString());
    this.applyPreference(enabled);

    if (!enabled) {
      // Clear all buffered metrics
      this.perfMon.clearAllMetrics();
      console.log('Telemetry disabled - all metrics cleared');
    }
  }

  private loadPreference(): boolean {
    const stored = localStorage.getItem(this.STORAGE_KEY);
    // Default to enabled for new users
    return stored === null ? true : stored === 'true';
  }

  private applyPreference(enabled: boolean): void {
    this.perfMon.setEnabled(enabled);
  }
}
```

#### Settings UI Component

```typescript
// features/settings/components/telemetry-settings/telemetry-settings.component.ts

@Component({
  selector: 'app-telemetry-settings',
  standalone: true,
  imports: [IonicModule, CommonModule],
  template: `
    <ion-list>
      <ion-item>
        <ion-label>
          <h2>Performance Monitoring</h2>
          <p>Help us improve the app by sharing anonymous performance data</p>
        </ion-label>
        <ion-toggle
          [checked]="telemetryEnabled()"
          (ionChange)="onToggle($event.detail.checked)"
        ></ion-toggle>
      </ion-item>

      <ion-item lines="none">
        <ion-note color="medium">
          <p><strong>What we collect:</strong></p>
          <ul>
            <li>Voice command response times</li>
            <li>Sync operation performance</li>
            <li>App load times</li>
            <li>Crash reports</li>
          </ul>
          <p><strong>We do NOT collect:</strong></p>
          <ul>
            <li>Personal information</li>
            <li>Job details or financial data</li>
            <li>Location data</li>
            <li>Voice recordings</li>
          </ul>
          <p>Data is stored securely in EU servers and automatically deleted after analysis.</p>
        </ion-note>
      </ion-item>
    </ion-list>
  `
})
export class TelemetrySettingsComponent {
  private readonly telemetrySettings = inject(TelemetrySettingsService);

  readonly telemetryEnabled = this.telemetrySettings.telemetryEnabled;

  onToggle(enabled: boolean): void {
    this.telemetrySettings.toggleTelemetry(enabled);
  }
}
```

### 2. Environment-Level Control (Build Configuration)

Disable monitoring entirely for development/testing environments.

```typescript
// ABOUTME: Environment configuration with telemetry control
// environments/environment.ts

export const environment = {
  production: false,
  telemetry: {
    enabled: false, // Disabled in dev
    sampleRates: {
      critical: 1.0,  // 100% in dev for testing
      secondary: 1.0  // 100% in dev for testing
    },
    uploadEnabled: false, // Never upload from dev
    localOnly: true       // Store locally only
  },
  // ... other config
};

// environments/environment.prod.ts
export const environment = {
  production: true,
  telemetry: {
    enabled: true,  // Enabled in production
    sampleRates: {
      critical: 0.1,  // 10% sampling
      secondary: 0.01 // 1% sampling
    },
    uploadEnabled: true,
    localOnly: false
  },
  // ... other config
};
```

#### Apply Environment Config in Service

```typescript
// Update PerformanceMonitoringService to respect environment config

@Injectable({ providedIn: 'root' })
export class PerformanceMonitoringService {
  private enabled = signal<boolean>(environment.telemetry.enabled);

  // Sampling rates from environment
  private readonly CRITICAL_SAMPLE_RATE = environment.telemetry.sampleRates.critical;
  private readonly SECONDARY_SAMPLE_RATE = environment.telemetry.sampleRates.secondary;

  setEnabled(enabled: boolean): void {
    // Only allow disabling, environment config sets the max
    if (!environment.telemetry.enabled) {
      this.enabled.set(false);
      return;
    }
    this.enabled.set(enabled);
  }

  recordCritical(name: string, value: number, unit: 'ms' | 'bytes' | 'count', tags?: Record<string, string>): void {
    if (!this.enabled()) return; // Early exit if disabled
    if (Math.random() > this.CRITICAL_SAMPLE_RATE) return;
    this.recordMetric(name, value, unit, tags);
  }

  async uploadMetrics(): Promise<void> {
    if (!environment.telemetry.uploadEnabled) {
      console.log('Metric upload disabled in environment config');
      return;
    }
    // ... rest of upload logic
  }

  clearAllMetrics(): void {
    this.metricsBuffer = [];
    this.clearLocalStorage();
  }

  private async clearLocalStorage(): Promise<void> {
    try {
      const db = await this.openMetricsDB();
      const tx = db.transaction('metrics', 'readwrite');
      await tx.objectStore('metrics').clear();
      await tx.complete;
      console.log('Local metrics storage cleared');
    } catch (error) {
      console.error('Failed to clear metrics storage:', error);
    }
  }
}
```

### 3. Tenant-Level Control (Business Settings)

Allow business owners to disable telemetry for their entire organization (e.g., for privacy-sensitive industries).

```typescript
// ABOUTME: Tenant-level telemetry control
// libs/mobile/data-access/src/lib/services/tenant-settings.service.ts

export interface TenantSettings {
  tenantId: string;
  telemetryEnabled: boolean;
  // ... other tenant settings
}

@Injectable({ providedIn: 'root' })
export class TenantSettingsService {
  private readonly firestore = inject(Firestore);
  private readonly perfMon = inject(PerformanceMonitoringService);
  private readonly store = inject(Store);

  async loadTenantSettings(tenantId: string): Promise<void> {
    const docRef = doc(this.firestore, `tenants/${tenantId}/businessProfile`);
    const snapshot = await getDoc(docRef);

    if (snapshot.exists()) {
      const data = snapshot.data();
      const telemetryEnabled = data.telemetryEnabled ?? true; // Default to enabled

      // Apply tenant-level preference
      if (!telemetryEnabled) {
        this.perfMon.setEnabled(false);
        console.log('Telemetry disabled at tenant level');
      }
    }
  }
}
```

### 4. Feature Flag Control (Remote Configuration)

Use Firebase Remote Config for dynamic control without app updates.

```typescript
// ABOUTME: Remote feature flag control for telemetry
// libs/mobile/data-access/src/lib/services/feature-flags.service.ts

import { Injectable, inject } from '@angular/core';
import { RemoteConfig, getValue, fetchAndActivate } from '@angular/fire/remote-config';

@Injectable({ providedIn: 'root' })
export class FeatureFlagsService {
  private readonly remoteConfig = inject(RemoteConfig);
  private readonly perfMon = inject(PerformanceMonitoringService);

  async initialize(): Promise<void> {
    // Set defaults
    this.remoteConfig.defaultConfig = {
      'telemetry_enabled': true,
      'telemetry_sample_rate_critical': 0.1,
      'telemetry_sample_rate_secondary': 0.01,
      'telemetry_upload_enabled': true
    };

    // Fetch remote values
    await fetchAndActivate(this.remoteConfig);

    // Apply flags
    this.applyTelemetryFlags();
  }

  private applyTelemetryFlags(): void {
    const enabled = getValue(this.remoteConfig, 'telemetry_enabled').asBoolean();

    if (!enabled) {
      this.perfMon.setEnabled(false);
      console.log('Telemetry disabled via remote config');
    }

    // Update sample rates dynamically
    const criticalRate = getValue(this.remoteConfig, 'telemetry_sample_rate_critical').asNumber();
    const secondaryRate = getValue(this.remoteConfig, 'telemetry_sample_rate_secondary').asNumber();

    this.perfMon.updateSampleRates({
      critical: criticalRate,
      secondary: secondaryRate
    });
  }

  /**
   * Emergency kill switch - call this to disable telemetry immediately
   */
  async emergencyDisableTelemetry(): Promise<void> {
    console.warn('EMERGENCY: Disabling telemetry via feature flag');
    this.perfMon.setEnabled(false);
    this.perfMon.clearAllMetrics();
  }
}
```

#### Update Service to Support Dynamic Sample Rates

```typescript
// Add to PerformanceMonitoringService

export class PerformanceMonitoringService {
  private criticalSampleRate = signal<number>(environment.telemetry.sampleRates.critical);
  private secondarySampleRate = signal<number>(environment.telemetry.sampleRates.secondary);

  updateSampleRates(rates: { critical?: number; secondary?: number }): void {
    if (rates.critical !== undefined) {
      this.criticalSampleRate.set(Math.max(0, Math.min(1, rates.critical)));
    }
    if (rates.secondary !== undefined) {
      this.secondarySampleRate.set(Math.max(0, Math.min(1, rates.secondary)));
    }
    console.log('Updated sample rates:', {
      critical: this.criticalSampleRate(),
      secondary: this.secondarySampleRate()
    });
  }

  recordCritical(name: string, value: number, unit: 'ms' | 'bytes' | 'count', tags?: Record<string, string>): void {
    if (!this.enabled()) return;
    if (Math.random() > this.criticalSampleRate()) return; // Use dynamic rate
    this.recordMetric(name, value, unit, tags);
  }

  recordSecondary(name: string, value: number, unit: 'ms' | 'bytes' | 'count', tags?: Record<string, string>): void {
    if (!this.enabled()) return;
    if (Math.random() > this.secondarySampleRate()) return; // Use dynamic rate
    this.recordMetric(name, value, unit, tags);
  }
}
```

### 5. Backend Kill Switch (Cloud Function)

Reject metric uploads at the server level if needed.

```typescript
// ABOUTME: Cloud Function with kill switch for metric ingestion
// functions/src/ingest-metrics.ts

import * as functions from 'firebase-functions';
import { getFirestore } from 'firebase-admin/firestore';

// Read kill switch from Firestore config document
async function isTelemetryEnabled(): Promise<boolean> {
  const db = getFirestore();
  const configDoc = await db.collection('system_config').doc('telemetry').get();

  if (!configDoc.exists) return true; // Default to enabled

  return configDoc.data()?.enabled ?? true;
}

export const ingestMetrics = functions
  .region('europe-west1')
  .runWith({
    timeoutSeconds: 60,
    memory: '256MB'
  })
  .https.onCall(async (data: MetricPayload, context) => {
    // Check kill switch first
    const telemetryEnabled = await isTelemetryEnabled();

    if (!telemetryEnabled) {
      console.log('Telemetry disabled - rejecting metrics');
      return { success: false, reason: 'telemetry_disabled' };
    }

    // Check authentication
    if (!context.auth) {
      throw new functions.https.HttpsError('unauthenticated', 'Must be authenticated');
    }

    // ... rest of ingestion logic
  });
```

#### Admin Tool to Toggle Kill Switch

```typescript
// ABOUTME: Admin utility to control telemetry kill switch
// admin-tools/toggle-telemetry.ts

import { getFirestore } from 'firebase-admin/firestore';
import * as admin from 'firebase-admin';

admin.initializeApp();

async function setTelemetryEnabled(enabled: boolean): Promise<void> {
  const db = getFirestore();

  await db.collection('system_config').doc('telemetry').set({
    enabled,
    updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    updatedBy: 'admin'
  });

  console.log(`Telemetry ${enabled ? 'ENABLED' : 'DISABLED'} globally`);
}

// Usage:
// node toggle-telemetry.js --disable
// node toggle-telemetry.js --enable

const args = process.argv.slice(2);
if (args.includes('--disable')) {
  setTelemetryEnabled(false);
} else if (args.includes('--enable')) {
  setTelemetryEnabled(true);
} else {
  console.log('Usage: node toggle-telemetry.js [--enable|--disable]');
}
```

### 6. Debug Mode (Development Override)

Force telemetry on/off for debugging purposes.

```typescript
// Add to PerformanceMonitoringService or as global utility

export class PerformanceMonitoringService {
  constructor() {
    // Check for debug override in URL or localStorage
    this.checkDebugOverrides();
  }

  private checkDebugOverrides(): void {
    // URL parameter: ?telemetry=false
    const urlParams = new URLSearchParams(window.location.search);
    const urlOverride = urlParams.get('telemetry');

    if (urlOverride === 'false') {
      console.warn('DEBUG: Telemetry disabled via URL parameter');
      this.enabled.set(false);
      return;
    }

    if (urlOverride === 'true') {
      console.warn('DEBUG: Telemetry force-enabled via URL parameter');
      this.enabled.set(true);
      this.criticalSampleRate.set(1.0);  // 100% sampling
      this.secondarySampleRate.set(1.0); // 100% sampling
      return;
    }

    // localStorage override for persistent debugging
    const localOverride = localStorage.getItem('DEBUG_TELEMETRY');
    if (localOverride === 'false') {
      console.warn('DEBUG: Telemetry disabled via localStorage');
      this.enabled.set(false);
    } else if (localOverride === 'true') {
      console.warn('DEBUG: Telemetry force-enabled via localStorage');
      this.enabled.set(true);
      this.criticalSampleRate.set(1.0);
      this.secondarySampleRate.set(1.0);
    }
  }
}

// Debug commands for browser console:
// localStorage.setItem('DEBUG_TELEMETRY', 'false'); // Disable
// localStorage.setItem('DEBUG_TELEMETRY', 'true');  // Force enable
// localStorage.removeItem('DEBUG_TELEMETRY');       // Clear override
```

## Control Priority Hierarchy

When multiple controls are in place, they apply in this priority order (most restrictive wins):

1. **Environment Config** - If `environment.telemetry.enabled = false`, telemetry is completely disabled
2. **Backend Kill Switch** - Server rejects uploads even if client sends them
3. **Tenant-Level Setting** - Business owner can disable for entire organization
4. **User Preference** - Individual users can opt out
5. **Remote Config** - Dynamic adjustments to sample rates
6. **Debug Override** - Developer tools for testing

### Example Decision Flow

```typescript
function shouldRecordMetric(): boolean {
  if (!environment.telemetry.enabled) return false;           // #1 Environment
  if (!serverKillSwitch) return false;                        // #2 Backend (checked on upload)
  if (!tenantSettings.telemetryEnabled) return false;         // #3 Tenant
  if (!userPreferences.telemetryEnabled) return false;        // #4 User
  if (!remoteConfig.telemetryEnabled) return false;           // #5 Remote
  if (debugOverride === false) return false;                  // #6 Debug

  return true; // All checks passed
}
```

## Emergency Shutdown Procedure

If telemetry causes issues in production:

### Step 1: Immediate Remote Kill Switch
```bash
# Update Firebase Remote Config
firebase remoteconfig:set telemetry_enabled=false
```

### Step 2: Backend Kill Switch
```bash
# Run admin script
node admin-tools/toggle-telemetry.js --disable
```

### Step 3: Client Hotfix (if needed)
```typescript
// Push emergency update to environment.prod.ts
export const environment = {
  telemetry: {
    enabled: false, // Hard disable
    // ...
  }
};
```

### Step 4: Clear User Data (GDPR Compliance)
```typescript
// Cloud Function to bulk delete metrics
export const deleteAllMetrics = functions
  .https.onCall(async (data, context) => {
    // Require admin authentication
    if (!context.auth?.token?.admin) {
      throw new functions.https.HttpsError('permission-denied', 'Admin only');
    }

    const db = getFirestore();

    // Delete all metrics across all tenants
    const batch = db.batch();
    const snapshot = await db.collectionGroup('performance_metrics').get();

    snapshot.docs.forEach(doc => batch.delete(doc.ref));
    await batch.commit();

    console.log(`Deleted ${snapshot.size} metric documents`);
    return { deleted: snapshot.size };
  });
```

## Rollout Plan

### Phase 1: MVP Monitoring (Weeks 1-4)

- Implement `PerformanceMonitoringService` with critical metrics only
- Deploy ingestion Cloud Function
- Set up manual BigQuery queries (no automated dashboards)
- **Goal**: Validate metrics collection works end-to-end

### Phase 2: Dashboard & Alerts (Weeks 5-8)

- Create Google Data Studio dashboard for key metrics
- Implement crash rate alert
- Add voice latency monitoring
- **Goal**: Daily dashboard review, catch critical regressions

### Phase 3: Optimization Loop (Week 9+)

- Run weekly performance review meetings
- Prioritize top 3 slow operations
- Correlate performance with user feedback
- **Goal**: Continuous improvement based on data

## References

- [Web Vitals](https://web.dev/vitals/)
- [Firebase Performance Monitoring](https://firebase.google.com/docs/perf-mon)
- [BigQuery Best Practices](https://cloud.google.com/bigquery/docs/best-practices-performance-overview)
- [Mobile Performance Optimization](https://web.dev/mobile/)

---

[← Previous: Performance Optimization](./performance-optimization.md) | [Back to Index](./index.md)
