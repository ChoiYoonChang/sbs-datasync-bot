# Phase 6: 고급 기능 및 안정성 - 구현 코드

## 개요
배치 모니터링, 알림, 오류 처리, 성능 최적화 기능을 구현합니다.

## 1. 실시간 모니터링

### 1.1 BatchMonitoringService.java
**파일 위치**: `/src/main/java/com/sbs/datasync/service/BatchMonitoringService.java`
```java
package com.sbs.datasync.service;

import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.explore.JobExplorer;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;

@Service
public class BatchMonitoringService {

    private final JobExplorer jobExplorer;
    private final NotificationService notificationService;

    // SSE 연결 관리
    private final List<SseEmitter> emitters = new CopyOnWriteArrayList<>();
    private final Map<Long, JobExecutionStatus> executionStatuses = new ConcurrentHashMap<>();

    public BatchMonitoringService(JobExplorer jobExplorer, NotificationService notificationService) {
        this.jobExplorer = jobExplorer;
        this.notificationService = notificationService;
    }

    // SSE 연결 추가
    public SseEmitter addEmitter() {
        SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);

        emitter.onCompletion(() -> emitters.remove(emitter));
        emitter.onTimeout(() -> emitters.remove(emitter));
        emitter.onError((ex) -> emitters.remove(emitter));

        emitters.add(emitter);

        // 연결 즉시 현재 상태 전송
        sendCurrentStatus(emitter);

        return emitter;
    }

    // 실시간 상태 브로드캐스트
    public void broadcastStatus(JobExecution jobExecution) {
        JobExecutionStatus status = createStatus(jobExecution);
        executionStatuses.put(jobExecution.getId(), status);

        String message = createStatusMessage(status);
        broadcastMessage(message);
    }

    private void broadcastMessage(String message) {
        List<SseEmitter> deadEmitters = new ArrayList<>();

        for (SseEmitter emitter : emitters) {
            try {
                emitter.send(SseEmitter.event()
                    .name("batch-status")
                    .data(message));
            } catch (IOException e) {
                deadEmitters.add(emitter);
            }
        }

        // 죽은 연결 정리
        emitters.removeAll(deadEmitters);
    }

    // 주기적 상태 체크 (30초마다)
    @Scheduled(fixedRate = 30000)
    public void monitorRunningJobs() {
        try {
            List<String> runningJobNames = jobExplorer.getRunningExecutions("dataSyncJob")
                .stream()
                .map(execution -> execution.getJobInstance().getJobName())
                .toList();

            // 실행 중인 작업들의 상태 업데이트
            for (JobExecution execution : jobExplorer.getRunningExecutions("dataSyncJob")) {
                JobExecutionStatus status = createStatus(execution);

                // 장기 실행 작업 감지 (2시간 이상)
                if (execution.getStartTime() != null) {
                    long duration = System.currentTimeMillis() - execution.getStartTime().getTime();
                    if (duration > 2 * 60 * 60 * 1000) { // 2시간
                        notificationService.sendAlert(
                            "장기 실행 배치 감지",
                            String.format("배치 작업 '%s'가 2시간 이상 실행 중입니다. (실행 ID: %d)",
                                execution.getJobInstance().getJobName(), execution.getId()),
                            NotificationService.AlertLevel.WARNING
                        );
                    }
                }

                broadcastStatus(execution);
            }

            // 시스템 리소스 모니터링
            monitorSystemResources();

        } catch (Exception e) {
            logger.error("배치 모니터링 중 오류 발생", e);
        }
    }

    private void monitorSystemResources() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();

        double heapUsedPercent = (double) heapUsage.getUsed() / heapUsage.getMax() * 100;

        if (heapUsedPercent > 90) {
            notificationService.sendAlert(
                "메모리 사용률 경고",
                String.format("힙 메모리 사용률이 %.1f%%입니다.", heapUsedPercent),
                NotificationService.AlertLevel.CRITICAL
            );
        }

        // CPU 사용률 체크 (간단 구현)
        OperatingSystemMXBean osBean = ManagementFactory.getOperatingSystemMXBean();
        if (osBean instanceof com.sun.management.OperatingSystemMXBean sunOsBean) {
            double cpuUsage = sunOsBean.getCpuLoad() * 100;
            if (cpuUsage > 80) {
                notificationService.sendAlert(
                    "CPU 사용률 경고",
                    String.format("CPU 사용률이 %.1f%%입니다.", cpuUsage),
                    NotificationService.AlertLevel.WARNING
                );
            }
        }
    }

    static class JobExecutionStatus {
        private Long executionId;
        private String jobName;
        private String status;
        private LocalDateTime startTime;
        private Long progress;
        private String message;

        // getters, setters...
    }
}
```

### 1.2 NotificationService.java
**파일 위치**: `/src/main/java/com/sbs/datasync/service/NotificationService.java`
```java
package com.sbs.datasync.service;

import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Service
public class NotificationService {

    private final JavaMailSender mailSender;
    private final RestTemplate restTemplate;
    private final NotificationConfig config;

    public NotificationService(JavaMailSender mailSender, RestTemplate restTemplate, NotificationConfig config) {
        this.mailSender = mailSender;
        this.restTemplate = restTemplate;
        this.config = config;
    }

    public void sendBatchCompleteNotification(Long executionId, String jobName, boolean success, Map<String, Object> stats) {
        String subject = String.format("[SBS DataSync] 배치 작업 %s - %s",
                                     jobName, success ? "성공" : "실패");

        StringBuilder content = new StringBuilder();
        content.append("배치 작업이 완료되었습니다.\n\n");
        content.append("작업명: ").append(jobName).append("\n");
        content.append("실행 ID: ").append(executionId).append("\n");
        content.append("상태: ").append(success ? "성공" : "실패").append("\n");
        content.append("완료 시간: ").append(LocalDateTime.now()).append("\n\n");

        if (stats != null) {
            content.append("처리 결과:\n");
            content.append("- 총 처리 건수: ").append(stats.get("totalCount")).append("\n");
            content.append("- 성공 건수: ").append(stats.get("successCount")).append("\n");
            content.append("- 오류 건수: ").append(stats.get("errorCount")).append("\n");
            content.append("- 소요 시간: ").append(stats.get("duration")).append("초\n");
        }

        // 이메일 발송
        if (config.isEmailEnabled()) {
            sendEmail(subject, content.toString(), config.getNotificationEmails());
        }

        // Slack 알림
        if (config.isSlackEnabled()) {
            sendSlackMessage(subject, content.toString());
        }
    }

    public void sendAlert(String title, String message, AlertLevel level) {
        String subject = String.format("[%s] %s", level.name(), title);

        // 이메일 발송
        if (config.isEmailEnabled() && shouldSendEmail(level)) {
            sendEmail(subject, message, config.getAlertEmails());
        }

        // Slack 알림
        if (config.isSlackEnabled() && shouldSendSlack(level)) {
            sendSlackMessage(subject, message);
        }
    }

    private void sendEmail(String subject, String content, List<String> recipients) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom(config.getFromEmail());
            message.setTo(recipients.toArray(new String[0]));
            message.setSubject(subject);
            message.setText(content);

            mailSender.send(message);
            logger.info("이메일 알림 발송 완료: {}", subject);

        } catch (Exception e) {
            logger.error("이메일 알림 발송 실패: {}", subject, e);
        }
    }

    private void sendSlackMessage(String title, String message) {
        try {
            Map<String, Object> payload = new HashMap<>();
            payload.put("text", title);
            payload.put("attachments", List.of(
                Map.of(
                    "color", "good",
                    "fields", List.of(
                        Map.of("title", "메시지", "value", message, "short", false)
                    )
                )
            ));

            restTemplate.postForObject(config.getSlackWebhookUrl(), payload, String.class);
            logger.info("Slack 알림 발송 완료: {}", title);

        } catch (Exception e) {
            logger.error("Slack 알림 발송 실패: {}", title, e);
        }
    }

    public enum AlertLevel {
        INFO, WARNING, CRITICAL
    }
}
```

## 2. 오류 처리 및 복구

### 2.1 BatchRetryService.java
**파일 위치**: `/src/main/java/com/sbs/datasync/service/BatchRetryService.java`
```java
package com.sbs.datasync.service;

import com.sbs.datasync.entity.BatchExecutionHistory;
import com.sbs.datasync.entity.BatchJobConfig;
import com.sbs.datasync.repository.BatchExecutionHistoryRepository;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.concurrent.CompletableFuture;

@Service
public class BatchRetryService {

    private final JobLauncher jobLauncher;
    private final BatchExecutionHistoryRepository historyRepository;
    private final NotificationService notificationService;

    public BatchRetryService(JobLauncher jobLauncher,
                           BatchExecutionHistoryRepository historyRepository,
                           NotificationService notificationService) {
        this.jobLauncher = jobLauncher;
        this.historyRepository = historyRepository;
        this.notificationService = notificationService;
    }

    @Retryable(
        retryFor = {Exception.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 60000, multiplier = 2) // 1분, 2분, 4분 간격
    )
    public JobExecution retryFailedJob(Long executionId, String reason) throws Exception {
        BatchExecutionHistory failedExecution = historyRepository.findById(executionId)
            .orElseThrow(() -> new IllegalArgumentException("실행 이력을 찾을 수 없습니다: " + executionId));

        if (failedExecution.getExecutionStatus() != BatchExecutionHistory.ExecutionStatus.FAILED) {
            throw new IllegalStateException("실패한 작업만 재실행할 수 있습니다.");
        }

        BatchJobConfig jobConfig = failedExecution.getJobConfig();

        // 재실행 파라미터 구성
        JobParameters retryParameters = new JobParametersBuilder()
            .addLong("jobConfigId", jobConfig.getJobConfigId())
            .addString("jobName", jobConfig.getJobName())
            .addString("retryReason", reason)
            .addString("originalExecutionId", executionId.toString())
            .addLong("retryTimestamp", System.currentTimeMillis())
            .toJobParameters();

        // Job 실행
        Job job = jobFactory.getJob("dataSyncJob");
        JobExecution execution = jobLauncher.run(job, retryParameters);

        logger.info("재실행 완료: 원본 실행 ID={}, 새 실행 ID={}", executionId, execution.getId());

        return execution;
    }

    @Async
    public CompletableFuture<Void> scheduleRetryForCriticalJobs() {
        // 최근 24시간 내 실패한 중요 작업들 조회
        LocalDateTime yesterday = LocalDateTime.now().minusHours(24);

        List<BatchExecutionHistory> failedExecutions = historyRepository
            .findFailedCriticalJobsSince(yesterday);

        for (BatchExecutionHistory execution : failedExecutions) {
            try {
                // 재실행 조건 체크
                if (shouldRetry(execution)) {
                    retryFailedJob(execution.getExecutionId(), "자동 재실행");

                    // 재실행 알림
                    notificationService.sendAlert(
                        "배치 자동 재실행",
                        String.format("실패한 배치 작업 '%s'를 자동으로 재실행했습니다.",
                                    execution.getJobConfig().getJobName()),
                        NotificationService.AlertLevel.INFO
                    );
                }
            } catch (Exception e) {
                logger.error("자동 재실행 실패: 실행 ID={}", execution.getExecutionId(), e);
            }
        }

        return CompletableFuture.completedFuture(null);
    }

    private boolean shouldRetry(BatchExecutionHistory execution) {
        // 재실행 조건 로직
        // 1. 3번 이상 실패하지 않았는지
        // 2. 마지막 실패가 1시간 이상 전인지
        // 3. 시스템 리소스 상태가 양호한지

        long failureCount = historyRepository.countFailuresByJobConfigSince(
            execution.getJobConfig().getJobConfigId(),
            LocalDateTime.now().minusHours(24)
        );

        return failureCount < 3 &&
               execution.getEndTime().isBefore(LocalDateTime.now().minusHours(1));
    }
}
```

## 3. 성능 최적화

### 3.1 BatchPerformanceOptimizer.java
**파일 위치**: `/src/main/java/com/sbs/datasync/service/BatchPerformanceOptimizer.java`
```java
package com.sbs.datasync.service;

import org.springframework.batch.core.StepExecution;
import org.springframework.stereotype.Service;

import javax.management.MBeanServer;
import javax.management.ObjectName;
import java.lang.management.ManagementFactory;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class BatchPerformanceOptimizer {

    private final Map<String, PerformanceMetrics> metricsCache = new ConcurrentHashMap<>();
    private final MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();

    public OptimizationResult optimizeChunkSize(StepExecution stepExecution) {
        String jobName = stepExecution.getJobExecution().getJobInstance().getJobName();
        PerformanceMetrics metrics = metricsCache.computeIfAbsent(jobName, k -> new PerformanceMetrics());

        // 현재 성능 지표 수집
        long readCount = stepExecution.getReadCount();
        long writeCount = stepExecution.getWriteCount();
        long duration = stepExecution.getEndTime().getTime() - stepExecution.getStartTime().getTime();

        double throughput = (double) readCount / (duration / 1000.0); // records per second

        // 이전 실행과 비교하여 최적 청크 크기 계산
        int currentChunkSize = getCurrentChunkSize(stepExecution);
        int optimizedChunkSize = calculateOptimalChunkSize(throughput, currentChunkSize, metrics);

        metrics.updateMetrics(throughput, currentChunkSize, duration);

        return new OptimizationResult(optimizedChunkSize, throughput, metrics.getRecommendation());
    }

    private int calculateOptimalChunkSize(double currentThroughput, int currentChunkSize, PerformanceMetrics metrics) {
        // 메모리 사용량 체크
        long usedMemory = getUsedHeapMemory();
        long maxMemory = getMaxHeapMemory();
        double memoryUsagePercent = (double) usedMemory / maxMemory * 100;

        // 메모리 사용률이 70% 이상이면 청크 크기 감소
        if (memoryUsagePercent > 70) {
            return Math.max(50, currentChunkSize / 2);
        }

        // 처리량이 증가 추세면 청크 크기 증가
        if (currentThroughput > metrics.getAverageThroughput() * 1.1) {
            return Math.min(1000, currentChunkSize + 50);
        }

        // 처리량이 감소 추세면 청크 크기 감소
        if (currentThroughput < metrics.getAverageThroughput() * 0.9) {
            return Math.max(50, currentChunkSize - 50);
        }

        return currentChunkSize;
    }

    @Scheduled(fixedRate = 300000) // 5분마다
    public void performanceAnalysis() {
        // 전체적인 성능 분석 및 권장사항 생성
        StringBuilder report = new StringBuilder();
        report.append("=== 배치 성능 분석 리포트 ===\n");

        for (Map.Entry<String, PerformanceMetrics> entry : metricsCache.entrySet()) {
            String jobName = entry.getKey();
            PerformanceMetrics metrics = entry.getValue();

            report.append(String.format("\n[%s]\n", jobName));
            report.append(String.format("평균 처리량: %.2f records/sec\n", metrics.getAverageThroughput()));
            report.append(String.format("권장 청크 크기: %d\n", metrics.getRecommendedChunkSize()));
            report.append(String.format("메모리 효율성: %s\n", metrics.getMemoryEfficiency()));
        }

        logger.info(report.toString());
    }

    private long getUsedHeapMemory() {
        try {
            return (Long) mBeanServer.getAttribute(
                ObjectName.getInstance("java.lang:type=Memory"),
                "HeapMemoryUsage"
            ).getUsed();
        } catch (Exception e) {
            return 0;
        }
    }

    static class PerformanceMetrics {
        private double averageThroughput = 0.0;
        private int recommendedChunkSize = 100;
        private long totalExecutions = 0;
        private double totalThroughput = 0.0;

        void updateMetrics(double throughput, int chunkSize, long duration) {
            totalExecutions++;
            totalThroughput += throughput;
            averageThroughput = totalThroughput / totalExecutions;

            // 권장 청크 크기 업데이트 로직
            if (throughput > averageThroughput) {
                recommendedChunkSize = Math.min(1000, chunkSize + 25);
            }
        }

        // getters...
    }

    static class OptimizationResult {
        private final int recommendedChunkSize;
        private final double currentThroughput;
        private final String recommendation;

        // constructor, getters...
    }
}
```

## 4. 프론트엔드 모니터링

### 4.1 RealTimeMonitor.tsx
**파일 위치**: `/frontend/src/components/RealTimeMonitor.tsx`
```tsx
import React, { useEffect, useState } from 'react';
import {
  Card,
  CardContent,
  Typography,
  LinearProgress,
  Grid,
  Alert,
  Box,
  Chip,
} from '@mui/material';
import {
  Timeline,
  TimelineItem,
  TimelineSeparator,
  TimelineConnector,
  TimelineContent,
  TimelineDot,
} from '@mui/lab';

interface MonitoringData {
  runningJobs: Array<{
    executionId: number;
    jobName: string;
    progress: number;
    startTime: string;
    status: string;
  }>;
  systemHealth: {
    cpuUsage: number;
    memoryUsage: number;
    diskUsage: number;
    status: 'healthy' | 'warning' | 'critical';
  };
  recentAlerts: Array<{
    id: number;
    message: string;
    level: 'info' | 'warning' | 'error';
    timestamp: string;
  }>;
}

export default function RealTimeMonitor() {
  const [monitoringData, setMonitoringData] = useState<MonitoringData | null>(null);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    // SSE 연결
    const eventSource = new EventSource('/api/batch/monitoring/stream');

    eventSource.onopen = () => {
      setConnected(true);
      console.log('실시간 모니터링 연결됨');
    };

    eventSource.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        setMonitoringData(data);
      } catch (error) {
        console.error('모니터링 데이터 파싱 오류:', error);
      }
    };

    eventSource.onerror = (error) => {
      setConnected(false);
      console.error('SSE 연결 오류:', error);
    };

    return () => {
      eventSource.close();
      setConnected(false);
    };
  }, []);

  if (!monitoringData) {
    return (
      <Card>
        <CardContent>
          <Typography>모니터링 데이터 로딩 중...</Typography>
        </CardContent>
      </Card>
    );
  }

  const getHealthColor = (status: string) => {
    switch (status) {
      case 'healthy': return 'success';
      case 'warning': return 'warning';
      case 'critical': return 'error';
      default: return 'default';
    }
  };

  return (
    <Grid container spacing={3}>
      {/* 연결 상태 */}
      <Grid item xs={12}>
        <Alert
          severity={connected ? 'success' : 'error'}
          variant="outlined"
        >
          실시간 모니터링: {connected ? '연결됨' : '연결 끊김'}
        </Alert>
      </Grid>

      {/* 실행 중인 작업들 */}
      <Grid item xs={12} md={6}>
        <Card>
          <CardContent>
            <Typography variant="h6" gutterBottom>
              실행 중인 작업 ({monitoringData.runningJobs.length})
            </Typography>
            {monitoringData.runningJobs.map((job) => (
              <Box key={job.executionId} mb={2}>
                <Box display="flex" justifyContent="space-between" alignItems="center" mb={1}>
                  <Typography variant="body2" fontWeight="medium">
                    {job.jobName}
                  </Typography>
                  <Chip
                    label={`${job.progress}%`}
                    size="small"
                    color="primary"
                  />
                </Box>
                <LinearProgress
                  variant="determinate"
                  value={job.progress}
                  sx={{ height: 6, borderRadius: 3 }}
                />
                <Typography variant="caption" color="textSecondary">
                  시작: {new Date(job.startTime).toLocaleString()}
                </Typography>
              </Box>
            ))}
            {monitoringData.runningJobs.length === 0 && (
              <Typography color="textSecondary">
                실행 중인 작업이 없습니다.
              </Typography>
            )}
          </CardContent>
        </Card>
      </Grid>

      {/* 시스템 상태 */}
      <Grid item xs={12} md={6}>
        <Card>
          <CardContent>
            <Box display="flex" justifyContent="space-between" alignItems="center" mb={2}>
              <Typography variant="h6">
                시스템 상태
              </Typography>
              <Chip
                label={monitoringData.systemHealth.status.toUpperCase()}
                color={getHealthColor(monitoringData.systemHealth.status)}
                size="small"
              />
            </Box>

            <Box mb={2}>
              <Box display="flex" justifyContent="space-between" mb={1}>
                <Typography variant="body2">CPU 사용률</Typography>
                <Typography variant="body2">
                  {monitoringData.systemHealth.cpuUsage}%
                </Typography>
              </Box>
              <LinearProgress
                variant="determinate"
                value={monitoringData.systemHealth.cpuUsage}
                color={monitoringData.systemHealth.cpuUsage > 80 ? 'error' : 'primary'}
              />
            </Box>

            <Box mb={2}>
              <Box display="flex" justifyContent="space-between" mb={1}>
                <Typography variant="body2">메모리 사용률</Typography>
                <Typography variant="body2">
                  {monitoringData.systemHealth.memoryUsage}%
                </Typography>
              </Box>
              <LinearProgress
                variant="determinate"
                value={monitoringData.systemHealth.memoryUsage}
                color={monitoringData.systemHealth.memoryUsage > 90 ? 'error' : 'primary'}
              />
            </Box>

            <Box>
              <Box display="flex" justifyContent="space-between" mb={1}>
                <Typography variant="body2">디스크 사용률</Typography>
                <Typography variant="body2">
                  {monitoringData.systemHealth.diskUsage}%
                </Typography>
              </Box>
              <LinearProgress
                variant="determinate"
                value={monitoringData.systemHealth.diskUsage}
                color={monitoringData.systemHealth.diskUsage > 85 ? 'error' : 'primary'}
              />
            </Box>
          </CardContent>
        </Card>
      </Grid>

      {/* 최근 알림 */}
      <Grid item xs={12}>
        <Card>
          <CardContent>
            <Typography variant="h6" gutterBottom>
              최근 알림
            </Typography>
            <Timeline>
              {monitoringData.recentAlerts.map((alert) => (
                <TimelineItem key={alert.id}>
                  <TimelineSeparator>
                    <TimelineDot color={
                      alert.level === 'error' ? 'error' :
                      alert.level === 'warning' ? 'warning' : 'primary'
                    } />
                    <TimelineConnector />
                  </TimelineSeparator>
                  <TimelineContent>
                    <Typography variant="body2" fontWeight="medium">
                      {alert.message}
                    </Typography>
                    <Typography variant="caption" color="textSecondary">
                      {new Date(alert.timestamp).toLocaleString()}
                    </Typography>
                  </TimelineContent>
                </TimelineItem>
              ))}
            </Timeline>
          </CardContent>
        </Card>
      </Grid>
    </Grid>
  );
}
```

**이유**:
- SSE로 실시간 데이터 스트리밍
- 시각적 시스템 상태 모니터링
- 알림 타임라인으로 이벤트 추적

## 5. 필요한 의존성 추가

### 5.1 build.gradle 업데이트
```gradle
dependencies {
    // 기존 의존성들...

    // 메일 발송
    implementation 'org.springframework.boot:spring-boot-starter-mail'

    // Quartz 스케줄러
    implementation 'org.springframework.boot:spring-boot-starter-quartz'

    // Retry 지원
    implementation 'org.springframework.retry:spring-retry'
    implementation 'org.springframework:spring-aspects'

    // 모니터링
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'

    // 비동기 처리
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
}
```

**이유**:
- Spring Mail로 이메일 알림 발송
- Quartz로 고급 스케줄링 기능
- Spring Retry로 자동 재시도
- Micrometer로 메트릭 수집

이상으로 Phase 6의 고급 기능 및 안정성 강화 구현 코드를 제시했습니다.